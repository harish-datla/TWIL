# The Depth Buffer

A per-pixel depth buffer stores the closest depth seen so far. Each fragment compares its depth with the stored value and is kept or discarded. Visibility therefore works independently of draw order, so opaque geometry need not be sorted back-to-front.

Depth is not physical distance. With Vulkan’s [0,1] range, near plane `n`, far plane `f`, and distance `z`:

```text
depth = f / (f − n) × (1 − n / z)
```

The `1/z` distribution places much more precision close to the camera. With near 0.1 m, far 1000 m, and a 24-bit buffer, depth reaches 0.5 at only 0.2 m. Far surfaces can round to the same value and z-fight.

## Reversed-Z

Use a floating-point depth buffer, map near to 1 and far to 0, clear to 0, and compare with GREATER. Float precision is denser near zero, which largely cancels the projection’s `1/z` distribution and makes distant surfaces more stable. Reversed-Z is separate from Vulkan’s [0,1] convention: it adds the near/far swap. Coplanar decals and equal depths can still fight.
