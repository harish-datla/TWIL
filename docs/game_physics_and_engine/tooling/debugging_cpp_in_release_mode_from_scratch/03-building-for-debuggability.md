# Building for debuggability without giving up speed

The first shipping build goes out stripped and without -g, and the first crash report comes back as a column of hex addresses nobody can name. The compiler flags that affect debuggability split into two groups: the ones that only emit debug info, which leave the generated code alone, and the ones that change the generated code, which can cost speed. Turn the first group on for any build you might want to debug, shipping builds included. The second group needs a deliberate trade-off.
Flag	Compiler	What it does	Codegen cost
-g / -g3	GCC, Clang	Emit DWARF debug info. -g3 additionally emits macro definitions.	None. Output is a separate set of .debug_* sections.
/Zi	MSVC	Emit CodeView debug info to a separate PDB.	None in the compiler. The linker's /DEBUG switches /OPT:REF and /OPT:ICF off and implies /INCREMENTAL, so release links pass /OPT:REF /OPT:ICF explicitly. /ZI (Edit and Continue) does change codegen; keep it out of shipping builds.
-gsplit-dwarf	GCC, Clang	Split debug info into .dwo files. Reduces link time and binary size on disk.	None.
-fno-omit-frame-pointer	GCC, Clang	Reserve rbp as the frame pointer in every function. Improves backtrace robustness when the unwind table is missing or wrong.	Usually under 1–2%, with outliers near 10%; one fewer general-purpose register[12].
/Oy-	MSVC (32-bit x86 only)	Disable frame-pointer omission. The x64 compiler has no /Oy option; x64 stack walks use the .pdata unwind tables instead.	Same trade-off as -fno-omit-frame-pointer, on x86.
-fasynchronous-unwind-tables	GCC, Clang	Emit .eh_frame at every instruction, not just at call sites.	Larger binary, no perf cost. On by default on x86-64 Linux.
-O2 -g / /O2 /Zi	both	The shipping-with-symbols build. Optimizations on, debug info on the side.	None beyond -O2 itself.
-Og	GCC, Clang	Optimize for debugging: enable optimizations that don't disturb stepping/locals.	Slower than -O2; faster than -O0; locals usually still inspectable.
-fno-strict-aliasing	GCC, Clang	Disable type-based alias analysis. Stops the compiler from assuming int* and float* point to disjoint storage.	Workload-dependent: blocks some load/store reordering and vectorization. In exchange, pointer type-punning behaves as written on GCC and Clang[13].

A reasonable default for in-development builds of an engine: -O2 -g -fno-omit-frame-pointer, with sanitizer flags enabled in the configurations that need them (covered in §8). Shipping builds keep -O2 -g and split the debug info to a symbol server. The real argument is over the frame pointer.

Fedora 38 (2023) and Ubuntu 24.04 (2024) switched their distribution packages to build with frame pointers on[12]. The benefit is that perf and any sampling profiler can build accurate stack traces by walking rbp at native speed; without it, perf has to either copy a chunk of stack on every sample and unwind it later with the DWARF tables (--call-graph=dwarf, expensive in overhead and output size) or use --call-graph=lbr, Intel's Last Branch Record hardware, which holds only 16 or 32 entries depending on the generation. Gregg reports the cost of keeping rbp reserved as usually under 1% in production, with reports of 1–2% elsewhere and outliers near 10%[12]. For a game that wants production-quality profiles, the trade is usually worth it; for a benchmark suite looking for the last 1%, it isn't.

Worked example

At 60 fps a frame has 16.7 ms. A 1% cost for keeping rbp reserved is 0.17 ms of that, and 2% is 0.33 ms. A 10% outlier would be 1.7 ms, which is why you measure your own engine before deciding. In exchange, every sample a profiler takes comes with a full call stack, collected by following the saved-rbp chain from frame to frame instead of copying stack memory out with each sample.

Apple's ARM64 platform ABI mandates the frame pointer[14]; you don't get a choice. Microsoft's x64 ABI doesn't require a frame pointer but does require the unwind tables, so stack walks work with or without one[9].
A note on asserts in shipping builds

The NDEBUG macro disables assert() from <cassert>. Many engines define a separate GAME_ASSERT that is kept on in shipping for the cheap ones (pointer non-null, range check, invariant) and stripped for the expensive ones. The cheap ones turn an undefined-behavior crash into a defined error log plus a minidump taken at the point the invariant broke, not wherever the bad state finally faulted.

The optimizer ladder below shows what each level of -O actually changes for a small loop. Pick a function, and the ladder lays out the assembly at each optimization level, from -O0 to the most aggressive rung:
Live · Optimizer ladder
sum_array
dot_product
contains
-O0 ~6 cyc/elt

sum_array:
  push  rbp
  mov   rbp, rsp
  mov   QWORD PTR [rbp-24], rdi    ; spill arg a
  mov   DWORD PTR [rbp-28], esi    ; spill arg n
  mov   DWORD PTR [rbp-4], 0       ; total = 0
  mov   DWORD PTR [rbp-8], 0       ; i = 0
.L_loop:
  mov   eax, DWORD PTR [rbp-8]     ; reload i from stack
  cmp   eax, DWORD PTR [rbp-28]    ; reload n from stack
  jge   .L_done
  mov   eax, DWORD PTR [rbp-8]     ; reload i again
  cdqe
  lea   rdx, [0+rax*4]
  mov   rax, QWORD PTR [rbp-24]    ; reload a from stack
  add   rax, rdx
  mov   eax, DWORD PTR [rax]
  add   DWORD PTR [rbp-4], eax
  add   DWORD PTR [rbp-8], 1
  jmp   .L_loop
.L_done:
  mov   eax, DWORD PTR [rbp-4]
  pop   rbp
  ret

-O1 ~1.5 cyc/elt

sum_array:
  test  esi, esi
  jle   .L_zero
  movsxd rsi, esi                  ; sign-extend n
  lea   rcx, [rdi+rsi*4]           ; end pointer
  xor   eax, eax                   ; total in register, not memory
.L_loop:
  add   eax, DWORD PTR [rdi]       ; one load per iter
  add   rdi, 4                     ; pointer-bump induction var
  cmp   rdi, rcx
  jne   .L_loop
  ret
.L_zero:
  xor   eax, eax
  ret

-O2 ~0.13 cyc/elt

sum_array:
  test       esi, esi
  jle        .L_zero
  cmp        esi, 8
  jb         .L_scalar               ; n < 8: scalar tail
  movsxd     rsi, esi
  vpxor      xmm0, xmm0, xmm0        ; vector accumulator = 0
  mov        rax, rdi
  lea        rcx, [rdi+rsi*4-32]
.L_vloop:
  vpaddd     ymm0, ymm0, [rax]       ; 8x int32 add per iter
  add        rax, 32
  cmp        rax, rcx
  jbe        .L_vloop
  vextracti128 xmm1, ymm0, 1         ; horizontal-add reduction
  vpaddd     xmm0, xmm0, xmm1
  ...                                 ; (reduce 4 lanes to 1)
  ...                                 ; scalar loop for the n % 8 tail
  vmovd      eax, xmm0
  vzeroupper
  ret

-O3 -funroll-loops ~0.06 cyc/elt

sum_array:
  ...                                 ; same prologue as -O2
  vpxor      xmm0, xmm0, xmm0
  vpxor      xmm1, xmm1, xmm1        ; 4 independent accumulators
  vpxor      xmm2, xmm2, xmm2
  vpxor      xmm3, xmm3, xmm3
.L_vloop:
  vpaddd     ymm0, ymm0, [rax+ 0]    ; 4 unrolled vector adds, no
  vpaddd     ymm1, ymm1, [rax+32]    ; cross-iteration latency chain
  vpaddd     ymm2, ymm2, [rax+64]
  vpaddd     ymm3, ymm3, [rax+96]
  add        rax, 128
  cmp        rax, rcx
  jbe        .L_vloop
  vpaddd     ymm0, ymm0, ymm1        ; combine accumulators
  vpaddd     ymm2, ymm2, ymm3
  vpaddd     ymm0, ymm0, ymm2
  ...                                 ; horizontal reduction + tail

mem-to-regCSELICMunrollingvectorizeunroll + vectorizefused-multiply-add
Pick a function at the top and read the rungs in order, from -O0 to the highlighted last one: the same source at rising optimization levels, each with a rough cycles-per-element estimate, and the tags below light up the transformations the last rung used. Output is approximate and trimmed (... marks elided prologue, tail and reduction code), in the style of Clang on x86-64 Linux with AVX2 enabled; without -mavx2 or -march=x86-64-v3 the vector code uses 16-byte xmm registers instead of ymm. Clang already splits the vector sum into several accumulators at -O2; the ladder shows that step on the last rung to separate it. Per-element cycle figures are rough steady-state estimates assuming the data is in L1; actual throughput depends on the microarchitecture, alignment, and surrounding code. The tags name standard LLVM transformations[15]: mem-to-reg (promoting stack slots to registers), common subexpression elimination, loop-invariant code motion, loop unrolling, loop and SLP autovectorization, and FMA contraction.

Worked example

Take the ladder's rough figures for sum_array over a million ints. At -O0, about 6 cycles per element is 6 million cycles, 2 ms on a 3 GHz core, an eighth of a 60 fps frame for one loop. At -O2, about 0.13 cycles per element is 130,000 cycles, roughly 0.04 ms. That gap of about 46× is the reason a debug build can't stand in for the one players run.
Check Your Understanding Q1 of 1

Which flag combination is the right default for a shipping game executable that must produce useful crash reports without sacrificing performance?
-O2 -g -fno-omit-frame-pointer (or /O2 /Zi on MSVC x64), with the debug info split out to a symbol server.
-O0 -g, since players can be told to enable a debug build if they hit a crash.
-O3 -fomit-frame-pointer -s (strip), to maximize speed and minimize size.
0/1 correct
