# Reading optimized assembly when the source is gone

A crash in a third-party DLL, a stripped vendored library, a driver fault, an inlined helper from a header you don't have: all of these end with you in front of a disassembly window with no source. The assembly tutorial covers the ISA itself; this section covers the workflow for inferring meaning from a code chunk you didn't write.

Five questions, in order, that turn a screen of hex into actionable information.
1. What function are you in?

The faulting RIP on the crash dump is an absolute address in a module that ASLR loaded at a random base. Subtract the module's load address (the dump's module list has it), then resolve the offset with the symbol table: addr2line -f -e game 0x2b14c7 on the GNU toolchain (for a position-independent ELF), the equivalent in llvm-symbolizer, or SymFromAddr from dbghelp.dll on Windows, which takes the absolute address once the module is loaded at the right base[21]. If the binary is stripped, the closest exported symbol is usually still resolvable: the dynamic-linker symbol table is separate from the debug-info one and is what the loader uses; strip by default leaves it. From the function name and the offset (+0x47) you can narrow disassembly to one function instead of the entire .text.

Worked example

A dump says the faulting RIP is 0x55d4c8ab14c7 and the game module was loaded at 0x55d4c8800000. The offset is 0x55d4c8ab14c7 − 0x55d4c8800000 = 0x2b14c7, the same on every machine and every run. If the symbol table says Game::Update starts at offset 0x2b1480, the crash is at Game::Update + 0x47 (0x2b14c7 − 0x2b1480 = 0x47), 71 bytes into the function.
2. What does the prologue tell you?

The first instructions of a function on x86-64 are a stereotyped prologue: zero or more push reg for callee-saved registers being preserved, then a sub rsp, N that allocates the local frame. The prologue tells you, before you read a single line of the body, how many callee-saved registers the function uses (r12–r15, rbx, rbp on System V[22]) and how much stack the function needs:
a typical non-leaf prologue, x86-64 System V

push  rbp                  ; save caller's frame pointer (only if -fno-omit-frame-pointer)
push  r15                   ; preserve callee-saved r15
push  r14                   ; preserve callee-saved r14
push  rbx                   ; preserve callee-saved rbx
sub   rsp, 0x48             ; allocate 72 bytes of frame
                            ; total stack used: 4 saves * 8 + 72 + 8 ret-addr = 112 bytes

The frame size puts an upper bound on the local data: 72 bytes of stack means at most ~9 8-byte locals or 18 4-byte ones, often less because the compiler aligns and pads. Three callee-saved registers (excluding the frame-pointer push of rbp) means the function did enough work to need three registers it couldn't clobber; a leaf function with no work would have skipped the prologue entirely. A frame size of 0x28 (40 bytes) on a function that calls another function is typical on Windows: 32 bytes of shadow space plus 8 bytes to keep rsp 16-byte-aligned at the next call[23].

Worked example

Add up the prologue above. Four pushes are 4 × 8 = 32 bytes, sub rsp, 0x48 is 72 more, and the call that got here pushed an 8-byte return address: 112 bytes in all. 112 is 7 × 16, and that's no accident. System V wants rsp 16-byte aligned at every call, the return address already took 8, so the pushes and the sub together have to come to 8 more than a multiple of 16: 32 + 72 = 104 = 6 × 16 + 8. When a frame size looks odd, alignment padding is the usual reason.
3. Where do the arguments live?

The first six integer arguments on System V are in rdi, rsi, rdx, rcx, r8, r9; the first four on Windows are in rcx, rdx, r8, r9[22][23]. The first eight floating-point arguments go in xmm0–xmm7 on System V; the first four in xmm0–xmm3 on Windows. A function whose body starts with mov rbx, rdi is preserving its first argument across a call. A function whose body starts with mov rax, [rdi] is dereferencing its first argument: that argument is a pointer, and the next reads tell you the struct shape.
4. What does each call site tell you?

A call rel32 at offset 0x47 in the function is calling some other function at a known offset; resolve that offset against the symbol table or the import table. A call qword ptr [rip + 0x...] is calling through a static function pointer: the address is in the GOT (Global Offset Table) on Linux or the IAT (Import Address Table) on Windows, both populated by the loader from the import metadata. A call qword ptr [rax + 0x18] right after mov rax, [rcx] (or [rdi]) is a virtual call: rax holds the vtable pointer loaded from the object's first 8 bytes, +0x18 selects slot 3 (0x18 is 24 bytes, and 24 / 8 = 3, counting from 0), and the slot index plus the class name from RTTI usually identifies the method.
5. Where do the data accesses point?

RIP-relative loads mov rax, [rip + 0x12345] are the standard way an optimized binary refers to globals, string literals, vtables, and constant data. The displacement is computed against the next instruction's address; a debugger or objdump will resolve them to the symbol they hit. Loads with a base+index addressing mode (mov eax, [rdi + rsi*4]) are array accesses; the base is usually a function argument (a pointer to the array), the index is the loop counter, and the scale is sizeof(element). With rdi = 0x2000 and rsi = 5, that load reads 0x2014, element 5 of an array of 4-byte ints.

The widget below applies the same workflow to a three-frame stack from a stripped binary. Pick a frame, or step with Prev and Next, and the panel on the right explains what that frame's prologue says about its size, where its arguments live, and how the walker finds the frame above it:
Live · Stack-frame walker
render_mesh
draw_node
scene_walk
◀ Prev frame
Next frame ▶
rsp+0x48 return address <- back to draw_node
rsp+0x40 saved rbp caller frame base
rsp+0x38 saved r15 caller r15
rsp+0x30 saved r14 caller r14
rsp+0x28 saved rbx caller rbx
rsp+0x20 spill: rdi (mesh*) 0x7f...12c0
rsp+0x18 spill: rsi (viewproj*) 0x7f...3a40
rsp+0x10 local: float4 view_pos ...
rsp+0x08 local: float4 view_pos ...
rsp+0x00 alignment padding 0x0000...

render_mesh(mesh, viewproj) · frame #0, where the fault happened. System V AMD64. The function takes two pointer arguments and calls into upload_uniforms.
What the prologue tells you

push rbp; push r15; push r14; push rbx means four callee-saved registers preserved. The sub rsp, 0x28 allocates 40 bytes: 32 for spilled args and a struct local, 8 for stack alignment.
Where the args live

rdi
    mesh* (1st arg)
rsi
    viewproj* (2nd arg)
r15
    holds mesh->vbo, set in body

Frame size

Total: 4 saves * 8 + 40 + 8 (return addr) = 80 bytes. The next frame starts at rsp + 80 after the epilogue.
Pick a frame and compare its stack rows on the left, highest address at the top, with the notes on the right. Each row is 8 bytes of stack memory. The callee-saved registers and the return address are recovered from the function's .eh_frame entry on Linux or its .pdata/.xdata entry on Windows[9]; this is the procedure debuggers and crash reporters use to walk a stack. The local-variable rows come from the DWARF location info (.debug_info, .debug_loclists) when the debug file is available; without it, "spill" rows are inferred from the prologue and the body's loads and stores.
