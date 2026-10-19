# Metallic-Roughness

glTF’s core material inputs map to BRDF values as follows:

```text
F0    = lerp(0.04, baseColor, metallic)
cdiff = baseColor × (1 − metallic)
alpha = roughness²
```

A red plastic has achromatic `F0=0.04`, colored diffuse, and a white highlight. A red metal has red `F0`, no diffuse body, and a red-tinted reflection. The 0.04 dielectric approximation corresponds to IOR 1.5: `((1.5−1)/(1.5+1))² = 0.04`.

Texture spaces are strict: base color and emissive are sRGB; metallic-roughness, normal, and occlusion are linear. In the common ORM packing, R is occlusion, G roughness, and B metallic. Sampling data textures through an sRGB view corrupts numeric values.

```cpp
struct MaterialFactors {
    glm::vec4 baseColorFactor;
    float metallicFactor;
    float roughnessFactor;
    float normalScale;
    float occlusionStrength;
    glm::vec4 emissiveFactor;
};
static_assert(sizeof(MaterialFactors) == 48);
```

Use a std140-friendly layout so CPU and shader offsets match.
