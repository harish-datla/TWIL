# Normals & Subnormals


A reverb tail fades, a damped spring settles, a velocity is multiplied by 0.98 every frame. Each one shrinks a value toward zero, and sooner or later it gets smaller than the smallest number the normal encoding can hold. What happens next matters for both correctness and speed.

Normal numbers have that implicit leading 1 and a stored exponent of 1..254. When a result gets too small to normalize (below 2−126 for binary32), the format doesn't jump straight to zero. It switches to subnormals (also called denormals): stored exponent 0 and an implicit leading bit of 0, which fill the gap down to zero with evenly spaced values. Precision drains away a bit at a time instead of all at once, which is called gradual underflow[1].

Worked example

Start at 1.0 and halve it over and over. After 126 halvings you're at 2−126 ≈ 1.18 × 10−38, the smallest normal float. Without subnormals the next halving would snap to zero.

With them, 23 more halvings still land on representable values, down to 2−149 ≈ 1.4 × 10−45, the smallest subnormal. The 150th halving lands exactly halfway between that and zero, and rounds to 0.
Trap:
FLT_MIN
is not the smallest float

std::numeric_limits<float>::min() (and Rust's f32::MIN_POSITIVE) is the smallest normal, about 1.18e-38. The smallest positive value is the smallest subnormal, denorm_min, about 1.4e-45, roughly seven orders of magnitude smaller[3]. And Rust's f32::MIN is the most-negative value, about -3.4e38. These three numbers are routinely confused.

Subnormals keep precision degrading gracefully toward zero instead of falling off a cliff. They have a performance story too, on some hardware a sharp one, which gets its own section at the end.

[Back to IEEE-754 Floating Point](index.md)
