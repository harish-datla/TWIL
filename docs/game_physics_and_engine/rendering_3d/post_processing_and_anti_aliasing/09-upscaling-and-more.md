# Upscaling and More

Temporal upscalers such as DLSS, FSR2, XeSS, and TAAU combine lower-resolution rendering with jittered history and motion vectors. Quality mode might render around 44% of 4K output pixels; performance mode around 25%. The missing detail is reconstructed from history, so disocclusion, transparency, and motion can ghost or shimmer.

DLSS uses a neural reconstruction; FSR2 is analytic. Their algorithm-specific claims should not be conflated, but both inherit TAA’s motion-vector and jitter requirements.

Motion blur and depth of field generally run in HDR before tone mapping so highlights retain energy. Motion blur gathers along velocity and can have silhouette artifacts. Depth of field derives a circle of confusion from depth and a thin-lens model. Vignette, chromatic aberration, and film grain usually run after tone mapping.
