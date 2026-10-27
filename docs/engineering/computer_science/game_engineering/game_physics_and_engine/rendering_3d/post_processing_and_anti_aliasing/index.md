# Post-Processing & Anti-Aliasing

The lighting pass produces HDR values far above 1.0, while displays show 0–1. This tutorial builds the back half of the frame: HDR full-screen passes, bloom, exposure, grading, tone mapping, and anti-aliasing. It focuses on a stable bloom pyramid and temporal AA with history management.

Time: ~55 min  
Level: Senior  
Prerequisites: [Deferred Rendering](../deferred_rendering_and_g_buffer/index.md) and [PBR](../pbr_materials_and_lighting/index.md).  
Stack: GLSL · C++ & Rust

## What you’ll be able to do

- Order HDR and LDR effects correctly.
- Build bloom with a downsample/upsample pyramid and suppress fireflies.
- Meter and smooth automatic exposure.
- Distinguish geometric, shading, and temporal aliasing.
- Implement TAA with Halton jitter, motion vectors, and history clamping.

## Sections

- [The Post Chain](01-the-post-chain.md)
- [Bloom](02-bloom.md)
- [Exposure](03-exposure.md)
- [Color Grading](04-color-grading.md)
- [The Anti-Aliasing Problem](05-the-anti-aliasing-problem.md)
- [MSAA](06-msaa.md)
- [FXAA and SMAA](07-fxaa-and-smaa.md)
- [TAA](08-taa.md)
- [Upscaling and More](09-upscaling-and-more.md)
- [Pitfalls](10-pitfalls.md)
- [What’s Next](11-whats-next.md)
