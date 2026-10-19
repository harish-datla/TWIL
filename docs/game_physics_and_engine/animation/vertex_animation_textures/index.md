# Vertex Animation Textures

Vertex Animation Textures (VAT) bake animation into texture pixels once and replay it in a vertex shader. Thousands of crowd characters, baked simulations, shatters, splashes, and particle meshes can share a static mesh and instanced draw with almost no per-instance CPU animation work.

Time: ~45 min  
Level: Junior to mid graphics / tech-artist  
Prerequisites: Basic HLSL or GLSL, UVs, and vertex shaders.  
Stack: HLSL · GLSL · C++ & Rust

## What you’ll be able to do

- Lay out vertices across a VAT texture and frames down it.
- Encode positions through a bounding box and choose storage precision.
- Decode soft-body VAT with normals and frame blending.
- Explain rigid-body, fluid, and sprite VAT.
- Choose VAT versus skeletal animation and budget texture memory.

## Sections

- [Why a VAT](01-why-a-vat.md)
- [A Short History](02-a-short-history.md)
- [The Core Idea](03-the-core-idea.md)
- [Texture Layout](04-texture-layout.md)
- [Encoding Positions](05-encoding-positions.md)
- [Decoding in the Shader](06-decoding-in-the-shader.md)
- [Normals](07-normals.md)
- [Frame Interpolation](08-frame-interpolation.md)
- [Soft-Body VAT](09-soft-body-vat.md)
- [Rigid-Body VAT](10-rigid-body-vat.md)
- [Fluid VAT](11-fluid-vat.md)
- [Sprite VAT](12-sprite-vat.md)
- [A Complete Shader](13-a-complete-shader.md)
- [VAT vs Skeletal Animation](14-vat-vs-skeletal-animation.md)
- [The Memory Wall](15-the-memory-wall.md)
- [Try It Yourself](16-try-it-yourself.md)
- [How Unreal Does It](17-how-unreal-does-it.md)
- [How Unity Does It](18-how-unity-does-it.md)
- [Pitfalls](19-pitfalls.md)
- [Where to Go from Here](20-where-to-go-from-here.md)
