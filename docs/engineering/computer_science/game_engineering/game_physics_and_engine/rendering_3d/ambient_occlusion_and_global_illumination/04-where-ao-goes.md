# Where AO Goes

AO modulates ambient or indirect light only:

```glsl
float ao = texture(ssaoBlurred, uv).r;
vec3 ambient = ambientIBL * albedo * ao;
vec3 color = directLighting + ambient;
```

Multiplying direct light by AO is the classic mistake. A contact under a bright lamp should still receive direct illumination; its ambient contribution is what nearby geometry reduces. Direct visibility has its own shadow-map term.

For example, with direct light 0.75, ambient 0.35, and AO 0.5, the correct result is `0.75 + 0.35×0.5 = 0.925`. Scaling everything gives `(0.75+0.35)×0.5 = 0.55`, incorrectly removing half the lamp’s light. Advanced PBR can use constrained specular occlusion or bent normals for indirect specular, but basic SSAO should affect ambient/IBL only.
