# Why Floats


A game world needs a bullet's position to the millimeter and a planet's orbit in the millions of kilometers, often in the same frame. A 32-bit float covers that dynamic range in a fixed number of bits, with arithmetic the CPU and GPU run in hardware. That range, not raw speed, is why games use them: integer add and multiply are usually just as fast, but integers can't cheaply hold fractions or span that range, and fixed-point trades the range away for uniform precision.

Worked example

Spend 32 bits on 16.16 fixed point (16 integer bits, 16 fraction bits) and every value is a multiple of 1/65,536 ≈ 0.0000153, with the largest just under 32,768. The step is the same everywhere, and anything past 32,768 doesn't fit.

A float spends the same 32 bits on relative precision instead: about 7 significant decimal digits at any size. Near 1.0 neighboring floats are 0.00000012 apart; near 20,000 they're 0.002 apart; the largest float is about 3.4 × 1038. If those are meters, a position 20 km out snaps to a 2 mm grid, and a camera-attached weapon hopping between grid points every frame is what you see as jitter.

The catch is that a float is an approximation with exactly defined rounding, and a handful of conventions decide how it rounds. Get those conventions wrong and you get the classic float bugs. Get them right and the bugs become predictable.

[Back to IEEE-754 Floating Point](index.md)
