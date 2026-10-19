# Signed vs Unsigned: SAR and SHR


A save-file loader reads a 32-bit field into a uint32_t and shifts it right to halve it. The field held −8. The result is 2,147,483,644 instead of −4. The shift did what it was told: the variable's type picked the wrong kind of shift.

Left shift has one behavior. Right shift has two, and the operand's type picks between them. A logical right shift always feeds zeros into the top bits (the x86 instruction is SHR). An arithmetic right shift fills the top bits with copies of the sign bit, so a negative number stays negative (the instruction is SAR).

Worked example

−8 in 32 bits is 0xFFFFFFF8: all ones except the bottom three bits.

Arithmetic shift right by 1: a copy of the sign bit (1) comes in at the top, giving 0xFFFFFFFC = −4.

Logical shift right by 1: a 0 comes in at the top, giving 0x7FFFFFFC = 2,147,483,644.

Shift −8 right by 31 arithmetically and every bit becomes a copy of the sign: 0xFFFFFFFF, which is −1. Do the same to +8 and you get 0.

In C and C++, >> on an unsigned type is logical. On a negative signed value it's implementation-defined in C (still, as of C23) and was in C++ until C++20. That's standardese for "every mainstream compiler copies the sign bit, but the standard won't promise it." C++20 requires the arithmetic shift for signed types[12]. On x86, compilers emit SAR for signed and SHR for unsigned[10]; ARM's equivalents are ASR and LSR. The listing below also uses the all-ones mask from the worked example for a branchless absolute value: XOR with the mask flips every bit of a negative x, and subtracting the mask (−1) adds the one, which is exactly "flip and add one".
The same bits, shifted two ways
C / C++
Rust

int32_t  signedVal   = -8;          // 0xFFFFFFF8
uint32_t unsignedVal = 0xFFFFFFF8u; // same 32 bits, read as unsigned

// Signed >> is arithmetic (SAR): the sign bit copies down.
signedVal   >> 1;   // -4   (0xFFFFFFFC)  divide-by-two stays correct

// Unsigned >> is logical (SHR): zeros come in at the top.
unsignedVal >> 1;   // 0x7FFFFFFC = 2147483644, not -4

// The sign-broadcast trick: -1 if negative, 0 if not.
int32_t signMask = signedVal >> 31;   // 0xFFFFFFFF (all ones)

// Branchless absolute value, from Hacker's Delight.
int32_t mask = x >> 31;             // 0 or -1
int32_t absX = (x ^ mask) - mask;   // flip-and-add-one only when negative
                                    // (overflows for INT_MIN, like abs())

The one-line takeaway

Convert or reinterpret a value as an unsigned type and shift right, and you get a logical shift, even if the bits stand for a negative number. Mixing signedness around a shift is one of the quieter ways to ship a bug. Pick the type that matches the meaning, then let the compiler pick SAR or SHR.

[Back to Bit Shifting from Scratch](index.md)
