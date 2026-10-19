# The Microfacet BRDF

Cook–Torrance models a surface as microscopic mirror facets. Roughness spreads their orientations, producing broad highlights; low roughness concentrates them. With half-vector `h = normalize(v+l)`:

```text
f_specular = D × F × G / (4 × (n·v) × (n·l))
```

- **D, GGX/Trowbridge-Reitz:** `alpha² / (π × ((n·h)²(alpha²−1)+1)²)`, with `alpha = roughness²`.
- **G, Smith with Schlick-GGX:** `G1(v) × G1(l)` for masking and shadowing.
- **F, Schlick:** `F0 + (1−F0)(1−(v·h))⁵`; reflectance rises toward 1 at grazing angles.

```glsl
const float PI = 3.14159265359;
float distributionGGX(float NdotH, float alpha) {
    float a2 = alpha * alpha;
    float d = NdotH*NdotH*(a2 - 1.0) + 1.0;
    return a2 / (PI * d * d);
}
float geometrySchlickGGX(float NdotX, float k) {
    return NdotX / (NdotX * (1.0 - k) + k);
}
float geometrySmith(float NdotV, float NdotL, float k) {
    return geometrySchlickGGX(NdotV,k) * geometrySchlickGGX(NdotL,k);
}
vec3 fresnelSchlick(float cosTheta, vec3 F0) {
    return F0 + (1.0-F0) * pow(clamp(1.0-cosTheta,0.0,1.0),5.0);
}
```

Floor `NdotV` at a small positive value such as `1e-4` to avoid silhouette division problems. Production shaders often fold G and the denominator into a visibility term, but the textbook form makes each factor clear.
