# SSAO: The Details

Useful SSAO needs four refinements:

- **Normal-oriented hemisphere:** avoids self-occlusion below flat surfaces.
- **Radius:** too small misses contact darkening; too large darkens unrelated geometry.
- **Bias and range check:** bias prevents flat-surface acne; a depth-distance weighting such as `smoothstep(0,1,radius/abs(Δz))` rejects distant foreground geometry and suppresses halos.
- **Rotation noise and blur:** tile a small noise texture to rotate the kernel per pixel, then blur the resulting noise. A depth-aware/bilateral blur avoids bleeding across silhouettes.

```cpp
for (int i = 0; i < 64; ++i) {
    glm::vec3 sample(rand01()*2-1, rand01()*2-1, rand01());
    sample = glm::normalize(sample) * rand01();
    float scale = float(i) / 64.0f;
    sample *= glm::mix(0.1f, 1.0f, scale * scale);
    kernel.push_back(sample);
}
```

At 1080p, 64 depth samples per pixel are about 133 million reads; half-resolution AO or temporal accumulation are common performance improvements.
