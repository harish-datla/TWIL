# Shadow Mapping

Lit characters look pasted onto the floor when nothing casts a shadow. Shadow mapping renders scene depth from a light’s point of view, then tests each camera-pass fragment against that depth. This tutorial covers the standard two-pass algorithm, Vulkan conventions, acne, peter-panning, PCF, cascaded and omnidirectional maps, and advanced alternatives.

Time: ~55 min  
Level: Senior  
Prerequisites: [Going 3D](../perspective_depth_and_meshes/index.md), [PBR](../pbr_materials_and_lighting/index.md), and 3D Math.  
Stack: C++ & Rust (Vulkan) · GLSL

## What you’ll be able to do

- Render and sample a directional- or spot-light shadow map.
- Explain acne and peter-panning and select appropriate bias strategies.
- Soften edges with PCF and comparison samplers.
- Set up cascaded shadow maps without seams or crawling.
- Choose between cube maps, PCSS, variance maps, and ray-traced shadows.

## Sections

- [The Two-Pass Idea](01-the-two-pass-idea.md)
- [The Light View-Projection](02-the-light-view-projection.md)
- [The Depth-Only Pass](03-the-depth-only-pass.md)
- [Sampling the Map](04-sampling-the-map.md)
- [The Bias Problem](05-the-bias-problem.md)
- [PCF](06-pcf.md)
- [Cascaded Shadow Maps](07-cascaded-shadow-maps.md)
- [Omnidirectional Shadows](08-omnidirectional-shadows.md)
- [Aliasing and the Tradeoff](09-aliasing-and-the-tradeoff.md)
- [Beyond Basic Shadow Maps](10-beyond-basic-shadow-maps.md)
- [Pitfalls](11-pitfalls.md)
- [What’s Next](12-whats-next.md)
