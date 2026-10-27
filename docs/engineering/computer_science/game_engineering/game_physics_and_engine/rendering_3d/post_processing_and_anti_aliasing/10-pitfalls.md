# Pitfalls

- **One wide blur for bloom:** Use a progressive downsample/upsample pyramid.
- **Firefly bloom:** Karis-average the first downsample.
- **Bloom after tone mapping:** Add light-energy bloom in HDR before the tone-map.
- **Exposure pumping:** Use log-average luminance and temporal smoothing.
- **MSAA for specular sparkle:** Prefilter normals/roughness or use temporal accumulation.
- **Jitter in motion vectors:** Remove projection jitter from velocity reconstruction.
- **More history to fix ghosting:** Clamp or reject stale history instead.
- **Treating upscalers as free resolution:** They trade reconstruction artifacts for quality per cost.
