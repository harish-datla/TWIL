# Why skeletal animation

An animator wants the character to walk, run, crouch and wave, each keyed at 30 poses a second. Authoring a separate mesh per animation frame doesn't scale (that's vertex animation, which suits crowds; see the VAT tutorial). Skeletal animation instead stores a small skeleton of joints, animates the joints over time, and computes each vertex's deformed position from the joints it's attached to. A 60-joint skeleton drives a 50,000-vertex character; you animate 60 transforms, not 50,000 positions.

Worked example

Store each joint's pose as translation (3 floats), rotation (a 4-float quaternion) and scale (3 floats): 60 joints × 10 floats × 4 bytes = 2,400 bytes per frame. The same frame as raw vertex positions is 50,000 × 12 bytes = 600,000 bytes, 250 times more. A one-second clip at 30 frames a second is 72 KB of joint data against 18 MB of positions.
The pipeline, top to bottom

Skeleton (a joint tree) → bind pose + inverse bind matrices (the mesh's rest state) → a per-vertex skinning matrix (animated joint × inverse bind) → a deformation method (linear blend or dual-quaternion) → animation clips (keyframes sampled over time) → blending (mixing clips) → the GPU (a matrix palette, skinned in the vertex shader). This tutorial builds it in that order: structure, then deformation, then time, then hardware.
