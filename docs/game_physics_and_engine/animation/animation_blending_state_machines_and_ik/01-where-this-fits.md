# Where this fits

Every frame, something has to decide what pose the character is in before the GPU can deform its mesh. The Skeletal Animation module ends with a pipeline: a per-joint local pose (each joint's rotation and offset relative to its parent) → walk the hierarchy to globals → build the matrix palette → skin the vertices. Everything in this module produces or modifies that local pose, then hands it to the same matrix palette, the array of per-joint matrices the vertex shader uses. The full per-frame pipeline:

pose source (state machine or motion matching) → layer blends → IK adjusts → globals → palette → skin

Each stage reuses the skeletal module's primitives: the local-space per-joint cross-fade (slerp rotations, lerp translation/scale), additive-as-delta, and the two-bone analytic IK. This module goes deeper on what that one previewed rather than re-deriving it.

Worked example

One frame for a soldier on a staircase: the state machine is 0.075 s into a 0.25 s walk-to-run cross-fade, so its pose is 30% run. A reload layer overrides the 24 upper-body joints of a 65-joint skeleton. Foot IK lowers the pelvis 15 cm so the downhill foot reaches its step, then bends both legs to plant the feet. Only then are the 65 globals computed, the 65 palette matrices built, and the mesh skinned.
