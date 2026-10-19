# Why a VAT

Skeletal animation evaluates clips, blends poses, solves IK, builds bone palettes, and uploads per-character state every frame. With a thousand pedestrians, CPU animation can exhaust the frame before the GPU does.

VAT bakes every vertex position at every frame into a texture. Runtime meshes are static; the vertex shader samples `(vertex, frame)` and applies the decoded position. Per-instance state is mostly a time offset, so one instanced draw can animate a crowd.

VAT is especially useful for distant crowds, baked cloth or fluid simulations, shatters, splashes, foliage, and flock effects—motions that are recordings rather than rules. It trades runtime flexibility for low CPU cost and reproducibility.
