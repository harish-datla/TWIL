# Position and Normals

Reconstruct position from screen UV and sampled depth. Build NDC X/Y from UV, use Vulkan depth directly as NDC Z, multiply by the inverse projection or inverse view-projection, then divide by W. Do not apply the OpenGL `depth*2-1` remap; Vulkan depth is already 0..1.

```glsl
float depth = texture(gDepth, uv).r;
vec4 ndc = vec4(uv * 2.0 - 1.0, depth, 1.0);
vec4 world = invViewProj * ndc;
world /= world.w;
```

Pack a unit normal with octahedral encoding to two channels:

```glsl
vec2 signNotZero(vec2 v) {
    return vec2(v.x >= 0.0 ? 1.0 : -1.0,
                v.y >= 0.0 ? 1.0 : -1.0);
}
vec2 octEncode(vec3 n) {
    n /= abs(n.x) + abs(n.y) + abs(n.z);
    vec2 e = n.xy;
    if (n.z < 0.0) e = (1.0 - abs(e.yx)) * signNotZero(e);
    return e;
}
vec3 octDecode(vec2 e) {
    vec3 n = vec3(e, 1.0 - abs(e.x) - abs(e.y));
    if (n.z < 0.0) n.xy = (1.0 - abs(n.yx)) * signNotZero(n.xy);
    return normalize(n);
}
```

Two 16-bit channels use 4 bytes and represent the full sphere without a hemisphere sign ambiguity.
