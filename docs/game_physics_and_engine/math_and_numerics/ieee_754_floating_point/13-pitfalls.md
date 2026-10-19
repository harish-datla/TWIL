# Pitfalls

Wrong epsilon tolerance
Treating FLT_EPSILON (gap at 1.0) as the rounding bound (half that), or as a usable tolerance far from 1.0.
Jitter far from origin
ULP grows with magnitude. Keep coordinates camera-relative, don't just reach for doubles.
NaN check that never fires
x == NAN is always false; use std::isnan / f32::is_nan (or x != x). And -ffinite-math-only deletes all of them.
Kahan sum reverts to naive
-fassociative-math / -ffast-math proves the compensation zero and removes it.
Replay desyncs on another PC
Differing compilers, FMA, x87-vs-SSE, or transcendentals. Doubles don't fix it.
Audio dropouts when quiet
Reverb tails generate subnormals; enable FTZ/DAZ per audio thread.
Smallest-float confusion
FLT_MIN is the smallest normal, not the smallest positive (that's denorm_min).
Type-punning UB
Reading float bits via (uint32_t*)&f is UB in C++; use std::bit_cast or memcpy[5].

[Back to IEEE-754 Floating Point](index.md)
