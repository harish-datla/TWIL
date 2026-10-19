# Catastrophic Cancellation


Two crates sit 10 cm apart in that town 20 km out, and the physics code subtracts their positions to get the gap between them. The answer comes back 0.4 mm short, a 0.4% error.

That's catastrophic cancellation: subtract two nearly-equal numbers that already carry rounding error and the matching leading digits cancel, leaving the old error as most of what's left. The subtraction itself is exact; it exposes error that was already there[1].

Worked example

The crates are at x = 20,000.1 and x = 20,000.2. Stored as floats on the 2 mm grid from §6, they become 20,000.099609375 and 20,000.19921875.

Their difference is exactly 0.099609375, and the subtraction made no rounding error at all. The true gap is 0.1, so the result is 0.39% off. The error came from storing the positions: each was off by under a millimeter out of 20 km, and after the subtraction that same sub-millimeter error is 0.39% of the answer. Computing the gap from positions kept small (relative to a nearby origin) avoids it.
Goldberg's quadratic-formula example


[Back to IEEE-754 Floating Point](index.md)
