# The Signed-Division Trap


A gameplay programmer writes offset >> 4 to divide a signed offset by 16. The tests pass, but QA reports a one-unit error that only happens west of the origin, where the offset is negative. For unsigned values, x / 16 and x >> 4 are identical. For signed values they aren't, and the bug survives review because the two expressions look interchangeable.

The difference is which way each one rounds. An arithmetic right shift rounds toward negative infinity: it floors. C and C++ integer division rounds toward zero: it truncates. For positive numbers both just drop the fraction, so they agree. For a negative number that doesn't divide evenly, flooring goes one further from zero than truncating does.

Worked example

−7 / 2 is −3.5. Truncating gives -7 / 2 = −3. Flooring gives -7 >> 1 = −4.

−1 / 2 is −0.5: the division gives 0, but -1 >> 1 is still −1, because 0xFFFFFFFF shifted right with the sign copied in is still 0xFFFFFFFF.

−8 / 2 is exactly −4, and both give −4. Off by one, only for negatives, only when the division isn't exact.

Sometimes the floor is what you want. If a tile map finds a pixel's 16-pixel tile with x >> 4, pixel −1 lands in tile −1, which is correct, while x / 16 would put it in tile 0 alongside pixels 0 to 15. The bug is swapping one for the other without noticing that the rounding changed.

The compiler knows this, which is why signed x / 8 in the widget above wasn't a single sar. Before shifting a negative number right by n, it adds a bias of 2n−1, which nudges the floor back up to the truncated answer[5][17]. It picks the bias without a branch, using the all-ones sign mask from §4 or a conditional move.

Worked example

The widget below starts at x = −9, n = 2 (divide by 4). −9 / 4 is −2.25.

Bare shift: -9 >> 2 = −3 (floored). C division: −2 (truncated).

With the bias: x is negative, so the bias is 22 − 1 = 3. (−9 + 3) >> 2 = −6 >> 2 = −2, matching C. For a non-negative x the bias is 0 and nothing changes.
Live · Round toward zero vs round toward −∞ 2^2 = 4
dividend x -9
shift n (divisor = 2ⁿ) 2

x >> n (floor)
    -3
x / 2ⁿ (C, toward 0)
    -2
bias added
    3
(x + bias) >> n
    -2

x >> n = -3 but x / 2ⁿ = -2. The shift rounded toward −∞. Adding the bias 3 fixes it.
The number line shows the dividend x (purple), the bare shift x >> n (red, below the line) and C's division (green, above). Drag x through the negatives: wherever x isn't a multiple of 2ⁿ the two markers sit one apart, and the readout shows the bias that closes the gap. Drag x positive and they merge. The corrected form (x + ((x >> 31) & (2ⁿ - 1))) >> n adds 0 to non-negative operands and 2ⁿ−1 to negative ones, reproducing C's toward-zero division with one extra add and one AND. MSVC emits exactly this for x / 8 on a signed int (cdq builds the sign mask, then and, add, sar); GCC and Clang add the same bias through a conditional move, as in the §5 widget.
Check Your Understanding Q1 of 1

A gameplay programmer writes camera.x = worldOffset >> 4 to divide a signed 32-bit offset by 16. QA reports the camera drifts by one unit, but only when the player is west of the origin (negative X). What happened?
The arithmetic right shift rounds toward negative infinity, while integer division rounds toward zero. For negative offsets that are not multiples of 16, the shift gives a result one lower than worldOffset / 16.
Right-shifting a negative signed value is one of the cases where the C standard lets anything happen, so the result is garbage west of the origin.
The shift overflowed because worldOffset was negative, wrapping the high bits.
0/1 correct

[Back to Bit Shifting from Scratch](index.md)
