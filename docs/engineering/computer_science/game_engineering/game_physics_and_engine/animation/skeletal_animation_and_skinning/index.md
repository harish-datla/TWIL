# Skeletal Animation & Skinning

Skeletal animation binds mesh vertices to a hierarchy of joints, animates the joints, and deforms the mesh with a skinning matrix. This tutorial covers the complete path from joint transforms and inverse bind matrices through GPU skinning, animation blending, glTF loading, and two-bone IK.

Time: ~55 min  
Level: Senior  
Prerequisites: Helpful, not required: the Going 3D tutorial and 3D Math.  
Stack: C++ & Rust · GLSL

## What you’ll be able to do

- Walk a joint hierarchy parent-first and compute model-space transforms.
- Build a correctly ordered skinning-matrix palette.
- Skin vertices with linear blend skinning and understand dual-quaternion skinning.
- Sample and blend animation clips with quaternion interpolation.
- Load a glTF skin and solve two-bone IK with the law of cosines.

## Sections

- [Why Skeletal Animation](01-why-skeletal-animation.md)
- [The Skeleton](02-the-skeleton.md)
- [The Inverse Bind Matrix](03-the-inverse-bind-matrix.md)
- [The Skinning Matrix](04-the-skinning-matrix.md)
- [LBS & DQS](05-lbs-and-dqs.md)
- [The Animation Clip](06-the-animation-clip.md)
- [Blending](07-blending.md)
- [The GPU Side & glTF](08-the-gpu-side-and-gltf.md)
- [IK & Root Motion](09-ik-and-root-motion.md)
- [Pitfalls](10-pitfalls.md)
- [What’s Next](11-whats-next.md)
