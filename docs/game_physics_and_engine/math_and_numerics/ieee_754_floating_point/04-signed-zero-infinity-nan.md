# Signed Zero, Infinity, NaN


A character's position turns into NaN, and on the next frame its skinned mesh disappears from the screen: every vertex multiplied by a NaN matrix is NaN too. The reserved exponents encode these special values, and each has a sharp edge.

Worked example

A steering function normalizes the direction to its target, and on this frame the target is exactly where the character stands. The direction is (0, 0, 0), its length is 0, and each component becomes 0/0 = NaN.

The velocity built from that direction is NaN as well, and the clamp if (speed > maxSpeed) speed = maxSpeed; on its length doesn't catch it, because every ordered comparison with NaN is false. One frame later the position is position + NaN = NaN, and so is every matrix built from it.

    Signed zero. +0.0 and −0.0 compare equal under == but have different bits and behave differently: 1.0 / +0.0 is +∞, 1.0 / −0.0 is −∞. Functions like copysign and atan2 tell them apart: atan2(+0.0, −1.0) is +π but atan2(−0.0, −1.0) is −π, so a heading computed from a velocity whose y is −0 reads −180° instead of +180°.
    Infinity. From overflow or division by zero. Arithmetic on it mostly stays infinite; ∞ − ∞ is NaN.
    NaN. From 0.0/0.0, ∞ − ∞, sqrt(−1). NaN is not equal to anything, including itself, which is exactly why x != x is the portable NaN test (use std::isnan / f32::is_nan).

Two NaN traps

First, x == NAN is always false; never test for NaN by comparing to it. Second, NaN is not a single bit pattern: any all-ones exponent with a nonzero fraction is a NaN, so there are millions of them. The top fraction bit distinguishes quiet from signaling NaN: a quiet NaN passes silently through arithmetic, while a signaling one is meant to raise an invalid-operation exception when used (on x86 and ARM, a top bit of 1 means quiet). The sign of a NaN is not portable, so don't rely on it[4].

[Back to IEEE-754 Floating Point](index.md)
