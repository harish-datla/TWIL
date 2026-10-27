# The GPU & the Graphics Pipeline

Your first textured quad warps along its diagonal, distant walls flicker through each other, and thin grass blades cost far more to shade than their pixel count suggests. These behaviors come from how the GPU turns triangles into pixels, and Vulkan makes every step explicit. This tutorial builds that mental model with a software rasterizer in C++ and Rust: GPU throughput, the triangle pipeline, rasterization, perspective-correct interpolation, depth, early-Z, and mobile tile-based rendering.

Time: ~55 min  
Level: Mid  
Prerequisites: The 3D Math tutorial is helpful, and you should be able to read C++ or Rust.  
Stack: C++ & Rust

## What you’ll be able to do

- Follow a vertex from the vertex shader to a framebuffer pixel.
- Decide triangle coverage with edge functions and barycentric weights.
- Fix texture warping with perspective-correct interpolation.
- Explain why thin triangles, discard, and distant surfaces cost performance or precision.
- Structure render passes so tile-based mobile GPUs retain their bandwidth advantage.

## Sections

- [Why a GPU Is Shaped This Way](01-why-a-gpu-is-shaped-this-way.md)
- [The Logical Pipeline](02-the-logical-pipeline.md)
- [Rasterization](03-rasterization.md)
- [The 2×2 Quad](04-the-2x2-quad.md)
- [Perspective-Correct Interpolation](05-perspective-correct-interpolation.md)
- [The Depth Buffer](06-the-depth-buffer.md)
- [Early-Z](07-early-z.md)
- [Immediate-Mode vs Tile-Based](08-immediate-mode-vs-tile-based.md)
- [The Baked Pipeline](09-the-baked-pipeline.md)
- [Pitfalls](10-pitfalls.md)
- [What’s Next](11-whats-next.md)
