# Going 3D: Perspective, Depth & Meshes

Load a 3D model and it comes out flat, upside down, inside-out, or with back faces showing through the front. This tutorial extends the Vulkan 2D renderer with perspective projection, a mandatory depth buffer, vertices with Z and normals, per-object matrices, glTF loading, camera transforms, and a scene graph.

Time: ~55 min  
Level: Senior  
Prerequisites: [Your First Triangle in Vulkan](../../rendering_2d/first_triangle_in_vulkan/index.md), 3D Math, [The GPU & the Graphics Pipeline](../../rendering_2d/gpu_and_graphics_pipeline/index.md), and [Textures, Samplers & Materials](../../rendering_2d/textures_samplers_and_materials/index.md).  
Stack: C++ (Vulkan/glm) · Rust (ash/glam)

## What you’ll be able to do

- Build a Vulkan perspective projection with the correct depth range and Y direction.
- Add, clear, and test a depth buffer in dynamic rendering.
- Load a glTF mesh and configure winding and culling.
- Feed per-object matrices without std140 layout corruption.
- Compute scene-graph transforms and update only dirty subtrees.

## Sections

- [From 2D to 3D](01-from-2d-to-3d.md)
- [Perspective Projection](02-perspective-projection.md)
- [The Camera and MVP](03-the-camera-and-mvp.md)
- [The Depth Buffer](04-the-depth-buffer.md)
- [The 3D Vertex](05-the-3d-vertex.md)
- [Loading a Mesh](06-loading-a-mesh.md)
- [Per-Object Data](07-per-object-data.md)
- [Culling and Winding](08-culling-and-winding.md)
- [The Scene Graph](09-the-scene-graph.md)
- [Pitfalls](10-pitfalls.md)
- [What’s Next](11-whats-next.md)
