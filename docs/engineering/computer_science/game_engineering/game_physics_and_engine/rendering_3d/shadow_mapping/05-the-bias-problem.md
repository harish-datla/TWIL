# The Bias Problem

Shadow acne is self-shadowing caused by one quantized depth value representing a finite, slanted texel footprint. Parts of the receiver fall behind the stored value and fail the comparison. It is worse when light grazes the surface.

Bias moves the comparison away from the surface. Too much creates peter-panning: the shadow detaches and floats. Bias is scene-, angle-, and resolution-dependent. Useful mitigations include slope-scaled depth bias (`vkCmdSetDepthBias`), which grows for edge-on polygons, and normal-offset bias, which offsets the receiver along its normal and reduces the visible gap.

For a flat ground under a light at elevation θ, a rough lower bound is:

```text
bias ≥ cot(θ) × texelSize / 2
```

With 5 cm texels, elevation 60° needs about 1.4 cm; at 30° about 4.3 cm; at 10° about 14 cm. A single magic constant cannot work across all angles and resolutions.
