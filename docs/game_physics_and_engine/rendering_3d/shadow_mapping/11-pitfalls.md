# Pitfalls

- **Remapping Z by `*0.5+0.5` on Vulkan:** Depth is already 0..1; remap only XY.
- **Calling acne a bias bug:** It is depth quantization over a texel; bias trades it against peter-panning.
- **Using one magic bias:** Scale it with slope, angle, map resolution, and cascade.
- **Perspective projection for a directional light:** Directional lights use orthographic projection; spots use perspective.
- **Averaging depths for soft edges:** PCF averages comparison results.
- **Cascades crawl:** Snap cascade bounds to texel increments.
- **Cascade seams:** Blend across split bands.
- **Back-face rendering everywhere:** Thin and single-sided geometry may cast no shadow.
