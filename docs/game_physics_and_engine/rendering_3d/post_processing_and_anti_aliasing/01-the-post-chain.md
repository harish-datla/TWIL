# The Post Chain

Post-processing is an ordered sequence of full-screen passes over the HDR lighting target. Ping-pong between two textures: each pass reads one and writes the other, never sampling the image it is currently rendering into.

Keep energy-linear effects such as bloom and exposure in HDR before tone mapping. Tone mapping compresses HDR into display range. FXAA, LDR grading, grain, and vignette commonly run after it. In Vulkan, the portable handoff is a separate pass plus an image barrier from color-attachment to shader-read-only layout; local reads expose only the current pixel and cannot replace neighbor-gathering passes.

A 1920×1080 RGBA16F target is about 16.6 MB. One full-resolution read/write pass moves at least 33 MB; five move about 166 MB per frame, or 10 GB/s at 60 fps. Run wide effects at reduced resolution and combine compatible cheap operations.
