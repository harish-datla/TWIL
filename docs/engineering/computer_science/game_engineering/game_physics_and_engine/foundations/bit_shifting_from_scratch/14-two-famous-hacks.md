# Two Famous Hacks


Two bit tricks earned their fame in games. The first reads a float's bits as an integer and does arithmetic on them; the second builds a random number generator from nothing but shifts and XORs.
The fast inverse square root

Lighting and physics normalize vectors constantly: divide a vector by its length to make it length 1. That's a multiply by 1/√x, where x is the squared length, and on late-1990s CPUs a division and a square root were both slow. The trick made famous by Quake III Arena (shipped 1999, source released 2005) replaces them with a shift, a subtraction and a few multiplies[19].

It starts by reading the float's 32 bits as an integer. (Quake does it with a pointer cast, which breaks C's strict-aliasing rule; memcpy or C++20's std::bit_cast is the defined way.) A float stored in the IEEE 754 format keeps its power-of-two exponent in the high bits, so that integer is roughly a scaled and offset base-2 logarithm of the float. Shifting it right by one halves it, which halves the logarithm, which takes the square root of the value. Subtracting the result from the constant 0x5f3759df negates the logarithm, turning √x into 1/√x, while the constant puts the offset back where it belongs. Read the bits back as a float and you have a first guess within about 3.4% of the true value. One Newton-Raphson step, a cheap correction that roughly doubles the number of correct digits, sharpens it to within about 0.18%. Chris Lomont worked out in 2003 why the constant is close to the best possible[8].

Worked example

x = 4.0 is stored as 0x40800000. Shift right by one: 0x20400000. Subtract from the constant: 0x5F3759DF − 0x20400000 = 0x3EF759DF.

Read that as a float: 0.483108. The true 1/√4 is 0.5, so the guess is 3.4% low.

One Newton step, y × (1.5 − 0.5 × x × y²), gives 0.499154: 0.17% low. These are the numbers the widget shows at its starting value.
Live · Float bits & the 0x5f3759df hack
01000000100000000000000000000000
signexponent (8)mantissa (23)
x = 4.0000

bits as int
    0x40800000
exponent field
    129 (unbiased 2)
i >> 1
    0x20400000 → exponent field roughly halved

raw approx
0.483108
error 3.378%
+1 Newton
0.499154
error 0.1693%
true 1/√x
0.500000
The bit row is x as a float, split into sign, exponent and mantissa, and the three boxes compare the bit-hack, the hack plus one Newton step, and the true 1/√x. Drag the slider and watch the exponent field move. The single right shift i >> 1 is the heart of it: halving the float's bits read as an integer roughly halves its exponent, which is how a shift can approximate a square root. The plot shows relative error for x from 0.25 to 16 on a log scale, with the top rule at 4%: the red curve is the bare bit-hack, the green curve is one Newton step later, and the purple line marks your current x. The red curve peaks just under 3.5%, and the green one stays under 0.2%. The error pattern repeats every factor of 4 in x, because the shift halves the exponent.
Xorshift: a PRNG that is only shifts

A particle emitter wants a few thousand random offsets a frame, and a replay system wants the same "random" loot rolls every time it plays back from the same seed. Both need a random number generator that is fast, tiny and deterministic. George Marsaglia's 2003 xorshift generators produce a stream of pseudo-random numbers from nothing but shifts and XORs[9]. Three lines, no multiply, no table: XOR the state with a shifted copy of itself, three times, with carefully chosen shift amounts.

With a well-chosen set of shift amounts such as (13, 17, 5), a 32-bit xorshift visits every non-zero 32-bit value, all 4,294,967,295 of them, before it repeats. (Zero is the one state it can't use: zero shifted and XORed with itself stays zero.) It's good enough for particle jitter, loot rolls and procedural placement, though it fails some stringent statistical tests and isn't suitable for cryptography. Scrambled variants like xorshift128+ improved the quality, and Vigna's xoshiro family has since superseded those in turn[21].

Worked example

The widget below seeds the state with 0x1A2B3C4D. One step of the three lines in its listing gives 0xC9F6F11C.

The widget plots it with the low 16 bits, 0xF11C = 61,724, as the column and the high 16 bits, 0xC9F6 = 51,702, as the row, each out of 65,535.
Live · Xorshift32 scatter state: 0x9B56633B
❚❚ pause
single step
reset

points plotted
    6,000

uint32_t x = seed;          // never zero
x ^= x << 13;
x ^= x >> 17;
x ^= x << 5;                 // x is the next output

Each point is one output of the generator: the low 16 bits pick its column and the high 16 its row, and the header shows the current state. Let it run and the plane fills evenly, with no visible stripes or clusters: three shifts and three XORs are enough to smear the state across the whole range. Press pause, reset and then single step to watch the 32-bit state change one iteration at a time; the first step gives 0xC9F6F11C.

[Back to Bit Shifting from Scratch](index.md)
