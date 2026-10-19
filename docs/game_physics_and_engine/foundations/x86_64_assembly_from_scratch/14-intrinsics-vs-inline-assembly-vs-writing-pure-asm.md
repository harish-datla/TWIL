# Intrinsics vs Inline Assembly vs Writing Pure asm


Your math library needs an instruction the language has no operator for: a fused multiply-add, a SIMD shuffle, a CPU feature query. There are three ways to drop below the C/C++ language barrier, in roughly decreasing order of how often you should use them:

    Compiler intrinsics. Header-defined functions (<immintrin.h> on x86, <arm_neon.h> on Arm) that mostly map to a single instruction each[17]. The compiler still allocates registers, schedules, and inlines. This is the default. Almost every shipping engine math library is intrinsics, not hand assembly.
    Inline assembly. GCC/Clang's asm statement[22] embeds asm in a C function with constraints describing the inputs, outputs, and clobbers. Useful for instructions your compiler exposes no intrinsic for, for exact instruction sequences the compiler must not rewrite, and occasionally for hand-tuned inner loops where the register allocator's choices are hurting. MSVC does not support inline asm in x64 code[23]; the Microsoft replacement is intrinsics or a separate .asm file compiled by ml64.
    Pure assembly files. .s on GCC/Clang via GAS, .asm on MSVC via MASM or NASM. Used for the lowest-level platform code: context switches, fiber and coroutine switching, and startup or signal-handling glue. Rare in application or game code.

CPUID, used at startup to detect available CPU features, makes a compact example. Production code should use the compiler's wrapper (__get_cpuid_count from <cpuid.h> on GCC and Clang, __cpuidex from <intrin.h> on MSVC); the raw inline-asm form shows how operand constraints work:
cpuid.c · inline asm with operand constraints

static inline void cpuid(int leaf, int subleaf,
                         int *eax, int *ebx, int *ecx, int *edx) {
  __asm__ (
    "cpuid"
    : "=a"(*eax),       // output: eax → *eax
      "=b"(*ebx),       // output: ebx → *ebx
      "=c"(*ecx),       // output: ecx → *ecx
      "=d"(*edx)        // output: edx → *edx
    : "a"(leaf),        // input:  leaf → eax
      "c"(subleaf)      // input:  subleaf → ecx (leaf 7, which reports AVX2 and
                        //         AVX-512, reads it; stale ecx gives wrong bits)
  );
}

The output constraints "=a", "=b", "=c", "=d" tell the compiler that cpuid writes those four registers and the values should go to the named C variables. The input constraints "a"(leaf) and "c"(subleaf) tell the compiler to load those values into eax and ecx before executing the instruction. That is the whole mechanism; the rest is constraint syntax[22].

Worked example

To test for AVX2, call cpuid(7, 0, &eax, &ebx, &ecx, &edx): leaf 7, subleaf 0. AVX2 support is bit 5 of ebx, so the test is (ebx >> 5) & 1. AVX-512 Foundation is bit 16 of the same register[10].

Two reliable failure modes. Forgetting clobbers. If your inline asm modifies a register that isn't an output, or touches memory the compiler can't see through the operands, it has to be declared in the clobber list ("memory" for the latter), or the compiler will assume nothing changed. Treating it as free. Inline asm is a black box to the optimizer: the compiler can't vectorize through it, schedule instructions across it, or constant-fold its inputs. A tight intrinsic-based loop is usually faster than the same loop with one inline-asm instruction in the middle.

[Back to x86-64 Assembly from Scratch](index.md)
