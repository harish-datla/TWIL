# 3D Math for Games

Your first triangle renders upside down, a model imported from another tool comes in mirrored with its faces culled, and a camera blend spins the long way round. The vectors, matrices and quaternions behind all three are the easy part. The bugs live in the conventions: row- or column-major, left- or right-handed, where the projection sends Y and how deep the clip space runs. This page builds the math in C++ and Rust, shows what each operation is for in an engine, and pins down the conventions Vulkan actually wants, so you can tell which convention a bug comes from and your first triangle comes out right-side up.
Time
~75 min
Level
Beginner to mid
Prereqs
You can read C++ or Rust. High-school vectors and trig. No graphics API needed; the conventions are built here.
Stack
C++ (glm) · Rust (glam)
◂ Build a Game Engine
Phase 2 · Math & numerics
Next · IEEE-754 Floating Point ▸
What you'll be able to do by the end

    Use dot and cross products for facing tests, view cones, normals and areas, and normalize without producing NaNs.
    Build model, view and projection matrices for Vulkan with glm or glam, and name which of four independent conventions a rendering bug comes from.
    Turn an axis and angle into a quaternion, rotate a vector with it, and explain when gimbal lock happens and why quaternions avoid it.
    Choose between SLERP and NLERP for a blend, and stop a rotation from turning the long way round.
    Compose and decompose TRS transforms, transform normals correctly under non-uniform scale, and pick a random-number generator that won't bias a loot table.


## Sections

- [The Conventions Are the Hard Part](01-the-conventions-are-the-hard-part.md)
- [Vectors](02-vectors.md)
- [Matrices](03-matrices.md)
- [Coordinate Spaces & Handedness](04-coordinate-spaces-and-handedness.md)
- [Model → View → Projection](05-model-view-projection.md)
- [Quaternions](06-quaternions.md)
- [Interpolating Rotations](07-interpolating-rotations.md)
- [Building & Decomposing Transforms](08-building-and-decomposing-transforms.md)
- [Random Numbers](09-random-numbers.md)
- [Pitfalls](10-pitfalls.md)
- [What's Next](11-whats-next.md)
