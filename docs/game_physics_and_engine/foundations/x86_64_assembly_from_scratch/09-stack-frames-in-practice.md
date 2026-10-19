# Stack Frames in Practice


A function that calls an SSE routine crashes with a misaligned load, but only when called from one particular caller. Stack alignment is the usual suspect, and reading the prologue tells you where it went wrong. The function-call stack on x86-64 grows downward: rsp decreases on a call or a push, increases on a ret or pop. A typical non-leaf function emits a prologue that allocates a frame, saves the registers it needs to preserve, and ends with an epilogue that reverses the prologue:
function_with_frame.s

my_function:
  ; ── Prologue ─────────────────────────────────────
  push rbp                  ; save the previous frame pointer
  mov  rbp, rsp             ; rbp now points at this frame's base
  push rbx                  ; save the callee-saved registers this function uses
  push r12
  sub  rsp, 0x30            ; 48 bytes of locals: return address + 3 pushes = 32 bytes,
                            ; so a multiple of 16 here keeps rsp 16-byte aligned

  ; ── Body ─────────────────────────────────────────
  ...

  ; ── Epilogue ────────────────────────────────────
  add  rsp, 0x30            ; deallocate locals
  pop  r12                  ; restore callee-saved registers in reverse
  pop  rbx
  pop  rbp
  ret

Worked example

Say rsp = 0x7FFC1008 on entry: 8 more than a multiple of 16, because call just pushed the 8-byte return address.

push rbp → 0x7FFC1000 (aligned). push rbx → 0x7FFC0FF8. push r12 → 0x7FFC0FF0 (aligned). sub rsp, 0x30 → 0x7FFC0FC0, a multiple of 16, so the body can call safely.

Had the frame been sub rsp, 0x28 instead, rsp would end 8 bytes off, and the first aligned SSE load in a callee could fault.

Three rules to remember when reading frames:

    The stack must be 16-byte aligned at every call site. Because call pushes the 8-byte return address, rsp on function entry is always 16k − 8. The prologue's push rbp brings it back to 16k. Each further push or local allocation has to leave rsp aligned again before the next call; that is why frame sizes often look like a multiple of 16 plus 8.
    The frame pointer is optional on x86-64. With -fomit-frame-pointer (default at -O1 and above in GCC and Clang for many years), the compiler skips push rbp; mov rbp, rsp and addresses locals off rsp directly. This frees a register but makes stack unwinding harder; profilers and debuggers fall back to DWARF CFI in .eh_frame, which is slower to walk. Fedora 38 (2023) and Ubuntu 24.04 (2024) reversed their distro defaults and now ship system binaries with frame pointers on, so that production profilers like perf can unwind through them cheaply[14]. Apple's AArch64 ABI requires the frame pointer on every non-leaf function[15].
    Simple leaf functions have no prologue at all. A function that calls nothing else and uses only caller-saved registers (like the add from §1) emits a single instruction body and a ret. On System V it can also use the 128-byte "red zone" below rsp for scratch storage without subtracting from rsp at all[6].

The red zone is the System V detail that catches Windows porters: a leaf function can legitimately have local_a at [rsp - 8], with rsp never adjusted. On Windows, memory below rsp may be overwritten at any time (by the OS or a debugger, for example), so the Microsoft ABI has no red zone and the compiler must sub rsp, ... for any local storage.

[Back to x86-64 Assembly from Scratch](index.md)
