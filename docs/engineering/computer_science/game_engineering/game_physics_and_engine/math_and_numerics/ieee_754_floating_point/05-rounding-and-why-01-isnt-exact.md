# Rounding, and Why 0.1 Isn't Exact


A spawn timer adds 0.1f every tick and fires when timer == 1.0f, but that check is never true. The reason is rounding: most results of float arithmetic land between two representable floats and have to snap to one of them.

IEEE-754's default rule is round-to-nearest, ties-to-even: snap to the nearest representable value, and on an exact halfway case pick the one whose last significand bit is 0[2].

Worked example

Above 16,777,216 (224) floats are 2 apart, so odd integers fall exactly halfway between two floats. 16,777,217 sits between 16,777,216 and 16,777,218 and rounds down to 16,777,216. 16,777,219 sits between 16,777,218 and 16,777,220 and rounds up to 16,777,220. Both go to the neighbor whose last bit is 0.
Trap: the default is banker's rounding, not round-half-up

Ties go to even, not always up. This removes the upward statistical bias that round-half-up would accumulate over millions of operations. The standard defines four other modes (toward zero, toward +∞, toward −∞, and ties-away, which is required only for decimal formats), but none is the default, and many library functions assume round-to-nearest-even, so changing the mode at runtime is fragile[2].

Rounding is also why 0.1 isn't exact. A fraction is representable in binary only if it's n / 2k. One tenth is 1/(2×5), and that factor of 5 makes it a non-terminating, repeating binary fraction, the same way 1/3 repeats in decimal. The nearest binary32 to 0.1 is slightly larger than one tenth[6].

Worked example

The stored 0.1f is exactly 0.100000001490116119384765625. Add it to a float timer ten times and each addition rounds again. The total is 1.0000001192, one step (2−23) above 1.0, so timer == 1.0f is false and the spawn never happens. Test timer >= 1.0f instead, or count whole ticks in an integer.

In double precision the famous 0.1 + 0.2 != 0.3 follows: three separate roundings, then a fourth on the addition, land on 0.30000000000000004. In binary32 the same errors happen to cancel and 0.1f + 0.2f == 0.3f holds, which is luck, not exactness. Integers, by contrast, are exact up to 224 in binary32 (253 in a double), so not everything is approximate, just any value that isn't a sum of powers of two within the precision budget.

[Back to IEEE-754 Floating Point](index.md)
