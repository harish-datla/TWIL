# SSAO: The Core Idea

Exact AO would cast many rays per pixel against the scene. Screen-space ambient occlusion uses the deferred renderer’s depth buffer. For each pixel, reconstruct view-space position, orient a sample volume around its normal, project samples into screen space, compare sampled depths, and count samples that land behind geometry.

If four of 16 samples beside a crate land inside the crate, occlusion is 0.25 and AO is 0.75. Open floor with all samples unblocked returns AO=1.

Because SSAO sees only the current depth buffer, it misses off-screen and hidden occluders, shifts as the camera moves, and can halo at depth discontinuities. Orienting the kernel to the normal hemisphere is essential: a full sphere treats the floor beneath a flat pixel as occluder and creates a gray cast.
