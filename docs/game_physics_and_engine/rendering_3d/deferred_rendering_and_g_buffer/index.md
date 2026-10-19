# Deferred Rendering & the G-buffer

Forward shading lights every object’s fragments once per light, including fragments later covered by nearer geometry. Deferred rendering draws geometry once into a G-buffer of screen-sized surface images, then lights each visible pixel once per light. It trades redundant shading for memory bandwidth and introduces decisions around MSAA, transparency, tiled culling, and clustered lighting.

Time: ~55 min  
Level: Senior  
Prerequisites: [Going 3D](../perspective_depth_and_meshes/index.md), [PBR](../pbr_materials_and_lighting/index.md), and [Shadow Mapping](../shadow_mapping/index.md).  
Stack: C++ & Rust (Vulkan) · GLSL

## What you’ll be able to do

- Fit a G-buffer into Vulkan 1.3’s guaranteed four color attachments.
- Reconstruct world position from depth and pack normals into two channels.
- Write MRT geometry and full-screen lighting passes with their barrier.
- Estimate bandwidth and explain deferred’s MSAA and transparency costs.
- Explain tiled and clustered light culling and Forward+.

## Sections

- [Why Forward Struggles](01-why-forward-struggles.md)
- [The Deferred Idea](02-the-deferred-idea.md)
- [The G-buffer](03-the-g-buffer.md)
- [Position and Normals](04-position-and-normals.md)
- [The Two Passes](05-the-two-passes.md)
- [Costs and Limits](06-costs-and-limits.md)
- [Tiled and Clustered](07-tiled-and-clustered.md)
- [Forward+ and the Verdict](08-forward-plus-and-the-verdict.md)
- [Pitfalls](09-pitfalls.md)
- [What’s Next](10-whats-next.md)
