# Pitfalls

- **Roughness not squared:** Use `alpha = roughness²` inside the BRDF.
- **Metal with a diffuse body:** Metals use colored `F0` and zero diffuse; apply `(1−metallic)`.
- **Normal map sampled as sRGB:** Sample direction data as linear/UNORM.
- **Wrong normal-map green channel:** Match +Y and −Y authoring conventions.
- **Metallic-roughness/occlusion as sRGB:** These are linear data textures.
- **Clamping instead of tone mapping:** Clamping clips highlights; use an HDR operator.
- **Double gamma:** Do not manually encode on top of an sRGB swapchain.
- **“PBR is physically correct”:** It is physically based; single-scatter models lose energy at high roughness.
