# The 2D Renderer: Sprite Batching

Drawing one textured quad is easy. Drawing ten thousand at 60 fps means a draw call per sprite can make the CPU the bottleneck while the GPU sits idle. This tutorial builds a Vulkan sprite batcher in C++ and Rust, then addresses atlas seams, dark cutout halos, transparent ordering, instancing, and text.

Time: ~55 min  
Level: Senior  
Prerequisites: [Your First Triangle in Vulkan](../first_triangle_in_vulkan/index.md), [Textures, Samplers & Materials](../textures_samplers_and_materials/index.md), and 3D Math.  
Stack: C++ (Vulkan) · Rust (ash)

## What you’ll be able to do

- Count when draw calls, rather than GPU work, determine frame rate.
- Build a per-frame vertex batcher that flushes only when necessary.
- Set up an orthographic 2D camera with the correct Y direction.
- Pack sprites into an atlas without seams or dark halos.
- Order transparent sprites while keeping batch count low.

## Sections

- [The Draw-Call Problem](01-the-draw-call-problem.md)
- [The Quad and Camera](02-the-quad-and-camera.md)
- [Sprite Batching](03-sprite-batching.md)
- [Instancing](04-instancing.md)
- [Atlas and Bleeding](05-atlas-and-bleeding.md)
- [The View-Projection](06-the-view-projection.md)
- [Alpha Blending](07-alpha-blending.md)
- [Draw Order](08-draw-order.md)
- [Text Rendering](09-text-rendering.md)
- [Pitfalls](10-pitfalls.md)
- [What’s Next](11-whats-next.md)
