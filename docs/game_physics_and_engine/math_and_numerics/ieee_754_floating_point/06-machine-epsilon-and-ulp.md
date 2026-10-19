# Machine Epsilon and ULP


An open-world game streams in a town 20 km from the world origin, and the player's gun starts to shimmer by a couple of millimeters every frame. Nothing changed in the code. What changed is the size of the numbers, because floats are not evenly spaced.

A ULP (unit in the last place) is the gap between one float and the next, and it doubles at every power of two, because the 23 fraction bits always split the range from one power of two to the next into the same 223 steps. Near 1.0 the gap is 2−23; just below 16,777,216 (224) the gap is 1.0, and above it, 2.0[1].

Worked example

To find the gap at a value, take the largest power of two at or below it, 2e, and divide by 223. In meters:

    At 1 m, e = 0: the gap is 2−23 ≈ 0.00000012 m, about a tenth of a micrometer.
    At 1,000 m, 29 = 512 ≤ 1,000 < 1,024, so e = 9: the gap is 2−14 ≈ 0.000061 m, 0.06 mm.
    At 20,000 m, 214 = 16,384 ≤ 20,000 < 32,768, so e = 14: the gap is 2−9 ≈ 0.00195 m, about 2 mm.

Every position in the town snaps to that 2 mm grid, and the gun shimmers as it hops between grid points.
Trap: there are two "machine epsilons"

People use one word, machine epsilon, for two different numbers that both describe float precision at 1.0, and the confusion causes wrong tolerance checks. FLT_EPSILON (which is what std::numeric_limits<float>::epsilon() and Rust's f32::EPSILON return) is 2−23 ≈ 1.19e-7, the gap between 1.0 and the next representable float, one ULP at 1.0[3][4]. The rounding-error bound (Goldberg's version) is half that, 2−24 ≈ 5.96e-8, because round-to-nearest is never off by more than half a step[1]. They differ by 2×. When you write a tolerance, know which one you mean.

The stepper walks one ULP at a time:
Magnitude 2^ 0
− 1 ULP
+ 1 ULP
jump to 2^24
The ticks are nine neighboring floats on a number line, centered on x (purple), with the gap to the next float marked "1 ULP" and the values printed below. Drag the magnitude slider or step with the buttons. At the default 1.0 the readout shows next = 1.0000001192092896 and 1 ULP = 1.1920928955078125e-7. Ticks sit at their true positions, so at an exact power of two the floats below are packed twice as densely as the floats above. Press "jump to 224": next = 16777218 and 1 ULP = 2, so the odd integers are gone. Step down one ULP to 16777215 and the gap is back to 1. A double only postpones this growth (and GPU pipelines run f32 in practice); the fix is to keep the values near the origin with local coordinate systems or camera-relative rendering.
Check Your UnderstandingQ1 of 2

What does std::numeric_limits::epsilon() return?
The gap between 1.0 and the next representable float, 2⁻²³ ≈ 1.19e-7.
The smallest positive float.
The maximum rounding error of any operation.
0/2 correct

Wrong answers, and why: epsilon() is the gap at 1.0, not the smallest float and not the rounding bound; and far-from-origin jitter is a magnitude problem that doubles fix or rounding don't actually solve.

[Back to IEEE-754 Floating Point](index.md)
