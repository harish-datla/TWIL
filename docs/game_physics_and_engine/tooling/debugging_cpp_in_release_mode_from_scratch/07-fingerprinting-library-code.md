# Fingerprinting library code

The top frame of a crash is an unnamed address inside the C runtime, and the code there is a tight loop of 32-byte vector loads and stores. You don't need its symbols to know it's a memcpy, and that the bug is almost certainly in whoever called it. Many of the disassemblies you'll find yourself reading are not your code. The C runtime, the C++ standard library, the platform allocator, and a handful of library functions show up in a large share of crash stacks and profiles. Knowing them by sight saves the time of resolving every faulting RIP back to a name. The patterns stay stable across compiler versions and look distinctive once you've seen them. Six worth memorizing:

    memcpy and similar. A short prologue, then a wide vector loop with paired AVX or SSE loads and stores: vmovdqu ymm0, [rsi + ...]; vmovdqu [rdi + ...], ymm0. On CPUs with ERMS, glibc switches to rep movsb above a size threshold, one instruction the CPU runs as a microcoded fast loop[24]. Either shape is a memcpy (or memmove).
    memset. Same shape as memcpy but store-only, from a register holding the fill byte broadcast across every lane: vmovd xmm0, esi; vpbroadcastb ymm0, xmm0; vmovdqu [rdi], ymm0, or rep stosb.
    strlen. An aligned-load loop followed by a SIMD compare against zero and a tzcnt or bsf on the result mask: pcmpeqb xmm0, xmm1; pmovmskb eax, xmm0; bsf ecx, eax. The pattern is "load, compare against zero, mask, bit-scan." It is the core loop of glibc's SSE2 strlen on x86-64; the AVX2 version does the same with ymm registers and tzcnt. For a 16-byte chunk holding "hello", a zero byte and ten nonzero bytes, the compare sets only byte 5, pmovmskb packs that into the mask 0x20 (bit 5), and bsf returns 5: the string is 5 bytes long.
    Vtable dispatch. mov rax, [rcx]; call qword ptr [rax + N] is a virtual call. rcx on Windows or rdi on System V is the this pointer; [rcx] reads the vtable pointer from offset 0; [rax + N] indexes into the vtable. The slot index is N / 8.
    Allocation site. A call to operator new(unsigned long) (mangled _Znwm on Linux, ??2@YAPEAX_K@Z under MSVC) or malloc directly, followed by the constructor, often inlined, writing fields through the returned pointer. A null check follows only malloc or the nothrow form of new; the throwing operator new never returns null. The mangled C++ symbol is the giveaway: _Z is the Itanium C++ ABI mangling prefix that GCC and Clang use[25].
    Stack-protector epilogue. A function compiled with -fstack-protector-strong ends with mov rax, [rsp + N]; sub rax, fs:[0x28] (older GCC uses xor) and a jne to a block that calls __stack_chk_fail on Linux. fs:[0x28] is glibc's copy of the stack canary in the thread control block; the compare is the canary check[26]. MSVC's /GS equivalent loads the cookie, XORs it with rsp, and calls __security_check_cookie.

The widget below is the kind of call you make in a real debugging session: a snippet of stripped assembly and three candidate functions. Pick the one that matches:
Live · Disassembly fingerprinter
; rdi = dst, rsi = src, rdx = n. n is in [16, 256] here.
prologue:
  cmp        rdx, 32
  jb         .small_tail
.vec_loop:
  vmovdqu    ymm0, [rsi]
  vmovdqu    [rdi], ymm0
  add        rsi, 32
  add        rdi, 32
  sub        rdx, 32
  cmp        rdx, 32
  jae        .vec_loop
memcpy
memset
std::copy
Round 1 of 3
Read the snippet, pick a function, then press Next snippet for the next of three rounds. The patterns are simplified to fit on screen; production glibc and MSVC CRT versions add aligned-head and tail handling around the same core loop. The three answer choices are a clean function and two near-misses; pick a near-miss and the explanation says why it doesn't fit.
