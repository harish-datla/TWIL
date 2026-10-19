# Aliasing and the Tradeoff

Even with correct bias, shadows can be blocky near the camera or smeared on surfaces nearly parallel to light rays. Perspective aliasing occurs when near-camera screen pixels map to too few shadow texels; cascades address it. Projective aliasing occurs when a receiver’s surface orientation stretches texels across many screen pixels.

A fixed-resolution map must choose between broad coarse coverage and narrow fine coverage. Higher resolution is not free: scattered accesses cost performance, and a tiny shadow such as a wire can be skipped when screen pixels sample between texels. Treat resolution, frustum fitting, cascades, filtering, and bias as related tradeoffs rather than expecting one setting to solve every artifact.
