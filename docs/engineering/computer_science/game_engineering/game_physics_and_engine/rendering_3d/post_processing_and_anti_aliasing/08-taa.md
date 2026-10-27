# TAA

Temporal AA jitters the projection by a different subpixel offset each frame and accumulates the result. Motion vectors reproject previous color into the current frame. An exponential blend often uses about 5–10% current color and the remainder history.

History is the hard part. Disocclusion and motion make reprojected samples stale, so clamp or clip history into the color AABB of the current 3×3 neighborhood, often in YCoCg. This suppresses ghosts but can soften the image; sharpening is commonly added afterward. Use a low-discrepancy sequence such as Halton for jitter, and keep jitter out of motion vectors.

```glsl
vec3 current = texture(currentColor, uv).rgb;
vec2 historyUv = uv - texture(motionVectors, uv).rg;
vec3 history = sampleCatmullRom(historyColor, historyUv);
ivec2 p = ivec2(gl_FragCoord.xy);
vec3 lo = current, hi = current;
for (int y=-1; y<=1; ++y) for (int x=-1; x<=1; ++x) {
    vec3 n = texelFetch(currentColor, p + ivec2(x,y), 0).rgb;
    lo = min(lo,n); hi = max(hi,n);
}
history = clamp(history, lo, hi);
float alpha = 0.1;
if (historyUv != clamp(historyUv, 0.0, 1.0)) alpha = 1.0;
outColor = mix(history, current, alpha);
```

More history is not a cure for ghosting; stale history must be rectified or rejected.
