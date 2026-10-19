# Autovectorization, Intrinsics, and When Each Wins


You've found a hot scalar loop and want it vectorized. Before spending a day on intrinsics, check whether the compiler already did it, or can with a hint. There are four ways to get vector code out of a C++ compiler, in roughly decreasing order of how often you should use them:

    Plain scalar loops, autovectorized. Write the obvious scalar code, compile with -O3 -march=native, look at the autovectorization report (-fopt-info-vec on GCC, -Rpass=loop-vectorize on Clang). The autovectorizer handles clean maps and reductions on contiguous arrays of trivially-copyable types. When it works, this is the right answer.
    Pragmas and attributes that nudge the autovectorizer. #pragma omp simd (enabled by -fopenmp-simd) or #pragma GCC ivdep declare that an inner loop has no loop-carried dependencies and the compiler may vectorize it[21]. __restrict declares that pointers don't alias. std::assume_aligned or __builtin_assume_aligned gives the compiler an alignment guarantee. The source stays scalar and readable while the output is vector code.
    Portable SIMD libraries. Google's Highway[22], std::simd in C++26[23], xsimd, SIMDe (which header-translates <immintrin.h> calls to NEON or WebAssembly SIMD[24]). The right answer when you want the explicit style of intrinsics but need it to run on x86, Arm, and the web. Engines ship their own versions of the idea: Unreal's VectorRegister4Float wrapper lowers to SSE or NEON per platform[12], and Unity's Mathematics package and Burst compiler[11] are a closely related design.
    Architecture-specific intrinsics. <immintrin.h> on x86, <arm_neon.h> on ARM. The form used throughout this tutorial. The right answer when the autovectorizer bailed out on the pattern you need, when you have to use an instruction the compiler can't pick from source (a specific pshufb table, an AVX-512 mask predicate), or when you need bit-identical results across architectures and the compiler's autovec choices vary by version.

The mental model: scalar loops are for the parts of the codebase that don't profile hot. Pragmas and portable wrappers are for the parts that do profile hot but aren't the absolute bottleneck. Intrinsics are for the kernels that sit near the top of the frame profile and have stayed there after every other optimization. Hand-written assembly sits below all four and is rare in engine code outside platform glue such as context switches.

A practical heuristic from Mike Acton, restated for SIMD specifically: look at the data and the access pattern first, write the inner loop second, switch to intrinsics third[13]. Most "the compiler won't autovectorize this" complaints are downstream of a layout problem the compiler isn't allowed to fix.

Check Your Understanding Q1 of 1

A hot loop sums an array: float total = 0; for (int i = 0; i < count; ++i) total += values[i];. Clang at -O3 -march=native reports loop not vectorized: cannot prove it is safe to reorder floating-point operations. What's the right fix?
Allow reassociation for this loop, with #pragma clang loop vectorize(enable), #pragma omp simd reduction(+:total), or -fassociative-math for the file, and accept that the result can change in the low bits.
Add __restrict to values. The compiler has to assume the array might alias total, and that aliasing assumption is what blocks the vector loop.
Compile with -O2 instead. The -O3 cost model rejects the loop, while -O2 uses a cheaper model that vectorizes it.
0/1 correct

[Back to SIMD from Scratch](index.md)
