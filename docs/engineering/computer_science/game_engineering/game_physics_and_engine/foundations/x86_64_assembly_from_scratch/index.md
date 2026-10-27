# x86-64 Assembly from Scratch

A hot loop runs at half the speed you expected. A release build crashes in a function the debugger can only show as instructions. A routine that works on Linux corrupts state on Windows. The answers are in the machine code the compiler produced, and this page teaches you to read it: the registers, how an instruction is encoded, the calling conventions that differ between Linux and Windows, stack frames, branches and branchless code, SIMD, atomics, and how the out-of-order core actually runs it all. Hardware claims are cited to vendor manuals, published measurements and papers.
Time
~75 min
Level
Engine programmer, intermediate to senior
Prereqs
You can read C or C++ comfortably. You know what a stack frame and a function pointer are. The Memory Model tutorial pairs naturally with §15 of this one.
Hardware
Any x86-64 machine from the last decade
◂ Build a Game Engine
Phase 0 · Foundations
Next · The C++ Memory Model ▸
What you'll be able to do by the end

    Read compiler output in Intel or AT&T syntax and map it back to the source: which register holds which argument, where the frame is, what each instruction does.
    Spot the System V vs Windows x64 differences that cause porting bugs: argument registers, callee-saved registers, shadow space, the red zone.
    Decode an instruction's bytes by hand, enough to read a crash dump from a stripped binary.
    Tell when a branch is nearly free and when branchless code wins, and why a loop with the same instruction count can run four times faster.
    Read SIMD code (lanes, packed and scalar forms, FMA) and the atomic instructions C++ std::atomic turns into.


## Sections

- [Why Assembly Still Matters When Nobody Writes It](01-why-assembly-still-matters-when-nobody-writes-it.md)
- [A Short History of the Architecture You're Reading](02-a-short-history-of-the-architecture-youre-reading.md)
- [The CPU You're Actually Programming For](03-the-cpu-youre-actually-programming-for.md)
- [The Register File](04-the-register-file.md)
- [Anatomy of an Instruction](05-anatomy-of-an-instruction.md)
- [The Core Instructions, by Frequency](06-the-core-instructions-by-frequency.md)
- [The lea Swiss Army Knife](07-the-lea-swiss-army-knife.md)
- [Calling Conventions: System V vs Windows x64](08-calling-conventions-system-v-vs-windows-x64.md)
- [Stack Frames in Practice](09-stack-frames-in-practice.md)
- [Reading Compiler Output: A Worked Vector Normalize](10-reading-compiler-output-a-worked-vector-normalize.md)
- [Branches, Mispredictions, and the Branchless Idiom](11-branches-mispredictions-and-the-branchless-idiom.md)
- [SIMD: SSE, AVX, AVX-512](12-simd-sse-avx-avx-512.md)
- [A Worked SIMD Example: 4x4 Matrix-Vector Multiply](13-a-worked-simd-example-4x4-matrix-vector-multiply.md)
- [Intrinsics vs Inline Assembly vs Writing Pure asm](14-intrinsics-vs-inline-assembly-vs-writing-pure-asm.md)
- [Atomics and Fences at the Machine Level](15-atomics-and-fences-at-the-machine-level.md)
- [μops, Ports, and Dependency Chains in Practice](16-muops-ports-and-dependency-chains-in-practice.md)
- [Pitfalls](17-pitfalls.md)
- [What's Next](18-whats-next.md)
- [Sources & Further Reading](19-sources-and-further-reading.md)
