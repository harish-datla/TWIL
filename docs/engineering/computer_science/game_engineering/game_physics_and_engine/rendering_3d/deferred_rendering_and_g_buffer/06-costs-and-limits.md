# Costs and Limits

Deferred’s main costs are:

- **Bandwidth:** Every G-buffer byte is written by geometry and read by lighting. A 16-byte layout costs about 4 GB/s at 1080p/60 before lights.
- **MSAA:** One surface per pixel cannot hold multiple coverage samples. Correct deferred MSAA needs per-sample G-buffers or edge-only per-sample shading. At 4×, 16 bytes becomes 64 bytes per pixel and about 133 MB at 1080p.
- **Transparency:** Translucency has multiple layers but the G-buffer stores one, so transparent objects need a forward pass after the deferred composite.
- **Material variety:** The lighting pass uses one material model; special skin, cloth, or car paint needs extra channels or material IDs and branches.

Few lights often favor forward because deferred’s fixed G-buffer cost has little to amortize. As light count and overdraw rise, deferred’s slower growth can win. The crossover depends on resolution, bandwidth, G-buffer format, and GPU.
