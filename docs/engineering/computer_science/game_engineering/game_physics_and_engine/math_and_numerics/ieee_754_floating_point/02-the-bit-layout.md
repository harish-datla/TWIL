# The Bit Layout


A debugger shows a position as 0x40D00000, or a network packet carries four bytes that should be a speed. To know what number those bits hold, and why some patterns mean infinity or NaN, you need the layout. A binary32 float (IEEE-754's 32-bit format, C++'s float and Rust's f32) is three fields in 32 bits: 1 sign bit, 8 exponent bits, and 23 fraction bits[1][2].

The sign says positive or negative. The fraction holds the digits. The exponent says where the binary point goes, the same job the power of ten does in 6.5 × 103. This formula turns the three fields back into the number, and it's what the hardware does every time it reads a float:
eq. 1 · binary32 value value = (−1)sign × 1.fraction × 2(exponent − 127)

In words: write a 1, a binary point, then the 23 fraction bits, which gives a number from 1 up to (not including) 2. Multiply it by 2 raised to the stored exponent minus 127, and flip the sign if the sign bit is set. The bias of 127 is what lets one unsigned 8-bit field encode both tiny and huge magnitudes. Hover any symbol to see what it stands for.

Worked example

Decode 0x40D00000. In binary that's 0 10000001 10100000000000000000000.

Sign bit 0: positive. Exponent field 10000001 = 129, and 129 − 127 = 2. Fraction 101 then zeros, so the significand is 1.1012 = 1 + 1/2 + 1/8 = 1.625. The value is 1.625 × 22 = 6.5.

The exponent is stored with a bias of 127, a fixed offset you subtract on the way out, so a stored 1..254 means an actual −126..+127. The leading 1. is implicit for normal numbers and not stored, which is why 23 stored fraction bits give 24 bits of precision. The 1 plus the fraction is the significand, the standard's word for what older texts call the "mantissa". Stored exponent 0 is reserved for zero and subnormals; stored exponent 255 for infinity and NaN (Not a Number, what 0/0 produces). binary64 (a double) uses the same scheme with 11 exponent bits (bias 1023) and 52 fraction bits.

The editor below shows the exact value of any binary32 and color-codes the three fields:
Presets:
1.0
0.1
π
∞
NaN
smallest subnormal
−0
The top row is the 32 bits: red sign, blue exponent, green fraction. Click any bit to flip it (or focus the bit row and use the arrow keys and Space), or start from a preset. The lines below decode it, and the last line is the exact value with no display rounding. Press 0.1: stored exponent 123 (2−4), significand ≈ 1.6000000, and exact = 0.100000001490116119384765625, a hair above one tenth. Zero the exponent field and the number goes subnormal (the readout says the implicit bit is now 0); set it to all ones and you're in infinity/NaN territory.

In code, C++20's std::bit_cast reads a float's bits without the undefined behavior of casting through a pointer[5], and Rust's to_bits / from_bits do the same[4]. Masks and shifts then pull the three fields apart:
Look at the bits
C++
Rust

// C++20: std::bit_cast is the UB-free way to reinterpret. (Reading a float
// through a uint32_t* breaks strict aliasing: undefined behavior.)
#include <bit>
uint32_t bits = std::bit_cast<uint32_t>(value);
float back = std::bit_cast<float>(bits);

uint32_t sign     = bits >> 31;
uint32_t exponent = (bits >> 23) & 0xFF;       // 8 bits
uint32_t fraction = bits & 0x7FFFFF;            // 23 bits


[Back to IEEE-754 Floating Point](index.md)
