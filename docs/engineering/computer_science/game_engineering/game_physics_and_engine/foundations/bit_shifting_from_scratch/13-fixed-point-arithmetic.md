# Fixed-Point Arithmetic


An RTS sends only player commands over the network, and every machine runs the same simulation from them. If one machine computes a unit's position as 10.000001 and another as 10.000002, the two games start to drift apart, and the gap only grows. One fix is to do the simulation's fractional math with integers, which give bit-identical results everywhere.

Fixed point stores a fraction as an integer with an imaginary binary point at a fixed position. A Q16.16 value is a 32-bit integer whose top 16 bits are the whole part and bottom 16 bits the fraction; the real number it stands for is that integer divided by 216 (65,536). Converting an integer n is a shift: n << 16. Addition and subtraction just work, because the points line up. Multiplication is where the shift earns its keep: multiplying two Q16.16 values gives a result scaled by 232, so you shift the product right by 16 to put the point back, using a 64-bit intermediate so the high bits don't fall off. Division shifts the other way.

Worked example

1.5 in Q16.16 is 1.5 × 65,536 = 98,304 (0x18000). 2.25 is 147,456 (0x24000).

Multiply the raw integers: 98,304 × 147,456 = 14,495,514,624. That's bigger than 232 (4,294,967,296), which is why the listing widens to 64 bits first.

Shift right by 16: 14,495,514,624 >> 16 = 221,184, and 221,184 / 65,536 = 3.375, which is 1.5 × 2.25.
Q16.16 fixed-point multiply
C / C++
Rust

using fixed = int32_t;            // Q16.16
constexpr int SHIFT = 16;

// The left shifts here rely on C++20 defining them for negative values.
// In C and older C++ that is UB (§7): multiply by 65536 there instead.
fixed fixed_from_int(int n) { return n << SHIFT; }

fixed fixed_mul(fixed a, fixed b) {
    // 32x32 -> 64 so the top bits survive, then shift the point back.
    int64_t wide = int64_t(a) * int64_t(b);
    return fixed(wide >> SHIFT);
}

fixed fixed_div(fixed a, fixed b) {
    // Pre-scale the dividend so the quotient keeps 16 fraction bits.
    int64_t wide = int64_t(a) << SHIFT;
    return fixed(wide / b);
}

Fixed point survives in 2026 because of that determinism. Floating-point results can differ across compilers, CPUs and optimization flags (a fused multiply-add here, a different rounding there), which quietly desynchronizes a lockstep simulation, where every client runs the same simulation from the same inputs and must get bit-identical results[18]. Integer math is exactly reproducible by construction, so some RTS netcode and replay systems still use fixed point rather than pin down every floating-point setting. It was also the practical option on consoles without a floating-point unit (FPU): the SNES and the Game Boy Advance did their world math in fixed point.
Live · Fixed-point number line
Q4.4
Q8.8
Q16.16
07
16
05
14
13
02
11
00
1.0
3.5
0.1
π

raw integer
    90 (0x5A)
real value
    5.6250
smallest step
    1/16 = 0.0625
split
    integer 0101 = 5 · fraction 1010 = 10/16

The row is the raw integer, with the fraction bits underlined in blue; the readout gives the real value it stands for and the smallest step the format can represent. Click bits to build a value, or press a preset. Load 0.1 and switch between formats (the widget re-rounds the value to each format's step): Q4.4 stores 2/16 = 0.125, Q8.8 stores 26/256 ≈ 0.1016, and Q16.16 stores 6554/65536 ≈ 0.100006, within 0.00001 but still not exact. 0.1 isn't a finite sum of powers of one half, so no binary format stores it exactly, fixed point or floating point. The integer/fraction boundary is just where you decide the point sits.

[Back to Bit Shifting from Scratch](index.md)
