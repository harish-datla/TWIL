# Why Assembly Still Matters When Nobody Writes It


Your physics step got 30% slower after a compiler upgrade, and the C++ didn't change. The only place the difference is visible is in the instructions the new compiler emitted. That's the job assembly does for most engine programmers today: it's how you find out what the machine is really being asked to do.

Very few production games ship hand-written assembly. Compilers produce good enough code for most hot paths, and a single ABI mismatch in hand-written code can corrupt a callee-saved register in a way that crashes three frames later. The reason to learn assembly in 2026 is not to write it. It is to read it. The compiler's output is the ground truth for what your CPU is being asked to do, and reading it answers questions that source-level reasoning can't.

Concrete situations where the disassembly is the only authoritative source:

    The optimizer didn't do what you assumed. A loop you thought was unrolled wasn't. A condition you thought compiled to cmov compiled to a branch. A small struct you thought was returned in registers got spilled to memory. Godbolt[1] exists for these moments: the assembly shows what the compiler actually did.
    A bug only reproduces under release optimizations. The compiler reordered or elided code in a way that exposed a latent UB. Reading the assembly is faster than printf-debugging a release build.
    You're integrating native libraries across an ABI. A C++ function calling into FFI code from Rust, an FMOD plugin, or a custom JIT, has to follow the platform's calling convention exactly. The only way to verify is to look at what each side emits.
    Microarchitecture tuning. When a hot SIMD loop is running at half throughput, the answer lives in the dependency chain or port pressure of the emitted instructions, not in the source.
    No symbols at the crash site. Reading a crash dump from a stripped binary in production means reading add rsp, 0x28; ret and inferring the function.

The CPU model in §3 is the standard mental picture engine programmers work from, and the SIMD walkthrough in §13 is a variant of the matrix-vector transform that renderers and math libraries implement[2]. By the end you'll also have the vocabulary (μops, ports, dependency chains, retirement) that the Intel and AMD optimization manuals[3][4] and Agner Fog's manuals[5] are written in.
A tiny disassembly to set the tone

Two lines of C. Compile with gcc -O2 on x86-64 Linux:
add.c

int add(int a, int b) {
  return a + b;
}

The compiler emits:
add.s · Intel syntax

add:
  lea   eax, [rdi + rsi]   ; eax = rdi + rsi (load-effective-address used as 3-op add)
  ret                      ; pop return address from [rsp], jump to it

Four lessons hiding in two instructions. First, the function arguments arrived in rdi and rsi because the System V ABI[6] puts the first two integer arguments there; the same code on Windows would have used rcx and rdx[7]. Second, the return value goes in rax (32 bits of which is eax). Third, the compiler used lea (Load Effective Address) as a three-operand non-destructive add: lea is the closest thing x86 has to an ARM-style add dst, src1, src2, and the compiler reaches for it constantly. Fourth, there is no stack frame at all, because nothing here required spilling state and the function is a leaf.

The rest of this page unpacks each of those four observations from first principles. By the end you will be able to look at a fifty-line dump and tell where the register allocator gave up, where a function was inlined, and which loop the compiler vectorized.
Check Your Understanding Q1 of 1

You compile int square(int x) { return x * x; } with gcc -O2 on x86-64 Linux. The function ends up emitting imul edi, edi; mov eax, edi; ret. Which observation is right?
edi holds the argument because the System V AMD64 ABI passes the first integer argument in rdi; the lower 32 bits are edi. The result is moved to eax because the same ABI returns 32-bit integer results in eax.
The mov eax, edi is redundant: a function can leave its result in the register that held its first argument, and the caller reads it from there.
The compiler used imul rather than mul because x is a signed int; an unsigned version of the function would compile to mul.
0/1 correct

[Back to x86-64 Assembly from Scratch](index.md)
