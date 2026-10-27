# HDR and Tone Mapping

Lighting can exceed 1.0 by orders of magnitude, while a display expects [0,1]. Render into a floating-point HDR target, apply exposure, tone-map into display range, then encode to sRGB.

Tone mapping is not clamping. Clamping makes every bright value flat white; an operator compresses highlights while preserving ordering and detail. Reinhard is:

```text
L_out = L / (1 + L)
```

For HDR values 0.25, 1, and 4, clamping gives 0.25, 1, 1; Reinhard gives 0.2, 0.5, 0.8. ACES-style fits provide a filmic look but are not neutral ground truth.

```glsl
vec3 color = hdrColor * exposure;
color = color / (color + vec3(1.0)); // Reinhard
// Only for a UNORM swapchain; omit if the swapchain format is sRGB.
color = pow(color, vec3(1.0 / 2.2));
```

The order is exposure → tone map → sRGB encode. An sRGB swapchain performs the final encoding in hardware, so adding a manual encode double-corrects the image.
