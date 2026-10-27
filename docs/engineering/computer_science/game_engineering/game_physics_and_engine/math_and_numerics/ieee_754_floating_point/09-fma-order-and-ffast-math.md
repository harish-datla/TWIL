# FMA, Order, and -ffast-math


A replay recorded on one build plays back differently on another build of the same source, on the same PC. Three facts about how float arithmetic gets evaluated explain most cases like this: fused multiply-add, evaluation order, and -ffast-math.

Fused multiply-add (FMA) computes a*b + c with a single rounding of the full-precision product-plus-sum, instead of rounding the product and then the sum[9]. It's more accurate for that expression and, on hardware with an FMA unit, often as fast as a bare multiply. C++ spells it std::fma; Rust spells it f32::mul_add.

Worked example

Take a = b = 1.000244140625 (that's 1 + 2−12) and c = −1. The exact product is 1.000488340854644775390625, which needs 25 bits. A float has 24, and the extra bit lands exactly halfway, so the rounded product is 1.00048828125.

Two roundings: 1.00048828125 − 1 = 0.00048828125. One rounding (FMA): 1.000488340854644775390625 − 1 = 0.000488340854644775390625, which a float holds exactly. The two answers differ by 0.012%, and FMA's is the exact one.
Fused multiply-add
C++
Rust

float r = std::fma(a, b, c);   // a*b + c, rounded once. Fast only if FP_FAST_FMAF is defined
                                  // (hardware FMA); otherwise a slow software routine.

The catch: a compiler may contract a plain a*b + c into a single FMA on its own, which changes the result versus the two-rounding form. "More accurate" is not "identical," and whether contraction happens depends on the compiler and flags, so the same source gives different numbers on different toolchains[9][10]. With the numbers above, the line return a*b + c; compiled by GCC 13 at -O2 -mfma becomes one vfmadd instruction and returns 0.000488340854644775390625; add -ffp-contract=off and it returns 0.00048828125.

Float addition is not associative. (a+b)+c can differ from a+(b+c) because each step rounds[1]. So summation order, SIMD lane-reduction order (SIMD runs one instruction on several values at once, then combines the lanes at the end), and multi-threaded partial-sum order all change the result. None of this is random: a fixed order is deterministic, and the answer changes only when the order does.

Worked example

a = 100,000,000, b = −100,000,000, c = 1, all exact as floats. (a + b) + c = 0 + 1 = 1. But a + (b + c) first computes −100,000,000 + 1, and floats near 108 are 8 apart, so that rounds back to −100,000,000, and the total is 0. Same three numbers, two orders, two answers.
Trap:
-ffast-math
trades IEEE behavior for speed, in ways that bite later

-ffast-math bundles several relaxations[10]. -fassociative-math lets the compiler reassociate, which is what deletes the Kahan compensation from §8. -ffinite-math-only lets it assume no NaN or infinity, so it can optimize isnan(x) to false and silently break NaN sentinels. -fno-signed-zeros makes x + 0.0 → x (wrong for −0.0). And on GCC and Clang (Linux), linking with -ffast-math or -Ofast adds a startup object (crtfastmath.o) that switches on flush-to-zero and denormals-are-zero for the whole process, changing the behavior of unrelated code; before GCC 13 this happened even when building a shared library, so merely loading that library changed results elsewhere. Audit it per translation unit; never blanket-enable it in a codebase that uses NaN sentinels, compensated sums, or lockstep determinism.

[Back to IEEE-754 Floating Point](index.md)
