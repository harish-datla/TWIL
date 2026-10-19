# Decoding in the Shader

The shader converts time to a frame coordinate, samples the normalized position, decodes it through the bounds, and applies the ordinary world-view-projection.

```hlsl
float frame = fmod(currentTime * bakeFramerate, frameCount);
float v = (frame + 0.5) / frameCount;
float3 encoded = positionTexture.SampleLevel(vatSampler,
                                              float2(vatLookup.x, v), 0).rgb;
float3 objectPosition = lerp(bboxMin, bboxMax, encoded);
output.clipPosition = mul(worldViewProjection,
                           float4(objectPosition, 1));
```

Use `SampleLevel(..., 0)` in HLSL because vertex shaders have no screen-space derivatives. GLSL can use `textureLod(..., 0.0)`. The texture should have no mipmaps and U should be centered on the vertex column. For robust clip looping, many production shaders point-sample and fetch/lerp the two frames explicitly.
