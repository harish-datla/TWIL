# A Complete Shader

A production soft-body shader should explicitly blend positions (and normals), decode bounds, wrap frame indices, and avoid depending on sampler filtering across clip boundaries.

```hlsl
float3 SamplePosition(float u, float frame) {
    float lo = floor(frame);
    float hi = fmod(lo + 1.0, frameCount);
    float a = frac(frame);
    float3 p0 = positionTexture.SampleLevel(pointSampler,
        float2(u, (lo + 0.5) / frameCount), 0).rgb;
    float3 p1 = positionTexture.SampleLevel(pointSampler,
        float2(u, (hi + 0.5) / frameCount), 0).rgb;
    return lerp(bboxMin, bboxMax, lerp(p0, p1, a));
}
```

Normals use the same two rows and decode with `normal*2−1`, then normalize. A complete implementation adds tangent-frame reconstruction for normal maps, animation-clip offsets, per-instance time offsets, rigid-body pieces, and world-space normal transformation. The core runtime remains a few texture reads and ordinary transforms.
