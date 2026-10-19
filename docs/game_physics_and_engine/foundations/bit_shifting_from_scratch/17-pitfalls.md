# Pitfalls


Use this as a review checklist: each item is a mistake that survives review because the code looks correct.

    Signed division is not a shift. x / 8 and x >> 3 diverge for negative, inexact values. Write the division and let the compiler add the bias.
    Shift count at or above the width is undefined. value << 32 on a 32-bit type is UB in C and C++, even though x86 masks it to a no-op and Java and JavaScript define the mask. Never let a count reach the width.
    Arithmetic versus logical is decided by the operand type. Shifting through an unsigned reinterpretation gives a logical shift even when the bits mean a negative number.
    Left-shifting a negative value, or into the sign bit, is undefined in C. (C++20 defines it; older C++ doesn't.) Cast to unsigned before packing values into the top bit, as in the RGBA pack from §11.
    The rotate idiom is UB at zero. (x << n) | (x >> (32 - n)) breaks when n is 0. Use std::rotl, or mask the complement.
    Shifting for speed is cargo-culting. The compiler strength-reduces multiplies and divides by constants. Hand-written shifts buy clarity about intent, not cycles.
    (1 << n) - 1 is undefined at the width. Building an all-ones mask of n bits is itself a shift, undefined when n equals the type width. Special-case it or use BZHI.
    Endianness is byte order in memory, not bit weight. Shifts are endianness-independent; only serializing the word to bytes involves endianness.
    PEXT/PDEP are slow on pre-Zen-3 AMD. Gate on vendor and family, not just the CPUID feature flag.
    Bit-scan of zero is undefined on the legacy instructions. BSR/BSF leave the result undefined for input 0; prefer LZCNT/TZCNT or guard the zero case.


[Back to Bit Shifting from Scratch](index.md)
