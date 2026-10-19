# Sampling the Map

The vertex shader outputs `lightViewProjection × worldPosition`. The fragment shader divides by W, maps X and Y to texture coordinates, and compares Z with the depth map.

OpenGL code often remaps all three NDC components with `*0.5+0.5`. Vulkan depth is already 0..1, so remap only X and Y. Z is used as-is.

```glsl
layout(set = 1, binding = 0) uniform sampler2DShadow shadowMap;

float shadowFactor(vec4 lightPos, vec3 N, vec3 L) {
    vec3 p = lightPos.xyz / lightPos.w;
    p.xy = p.xy * 0.5 + 0.5;       // XY only; Vulkan Z already 0..1
    if (p.z > 1.0) return 0.0;     // outside far plane: lit
    float bias = max(0.0015 * (1.0 - dot(N,L)), 0.0005);
    float lit = 0.0;
    vec2 texel = 1.0 / vec2(textureSize(shadowMap, 0));
    for (int x=-1; x<=1; ++x)
    for (int y=-1; y<=1; ++y)
        lit += texture(shadowMap,
            vec3(p.xy + vec2(x,y)*texel, p.z - bias));
    return 1.0 - lit / 9.0;
}
```

Use a clamp-to-border sampler with opaque-white border for X/Y outside the map, so those samples are lit. Guard Z beyond the far plane explicitly.
