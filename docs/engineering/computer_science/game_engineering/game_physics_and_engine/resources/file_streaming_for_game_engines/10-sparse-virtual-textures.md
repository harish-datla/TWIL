# Sparse Virtual Textures


You want a whole valley painted with one unique, hand-detailed texture, and at full resolution it would be 64 GB. Up to now "tile" has been an abstract unit. For textures specifically, there's a trick that turns choosing a mip level for every pixel on screen (a mip is one of a texture's precomputed, progressively halved copies) into a streaming problem. The trick is Sparse Virtual Textures (SVT), and Sean Barrett's GDC 2008 talk[1] is where it crystallized. id Tech 5 shipped it commercially as MegaTexture in Rage[2], and most major engines now have a variant (Unreal's is in §15).
The idea, in three pieces

    One huge logical texture. Pretend you have a 128k × 128k texture for the whole world. It would be 64 GB at 4 bytes per pixel; obviously it doesn't fit in VRAM.
    A small physical cache. A real GPU texture, maybe 4096 × 4096, divided into 64 × 64-texel tiles (16 KB each at 4 bytes per texel). This is what's actually resident.
    An indirection table. A small lookup texture (2048 × 2048 pixels, one pixel per logical tile) that maps logical tile coordinates to physical-cache coordinates. The shader samples the indirection texture, then samples the physical cache at the offset it found.

Worked example

A pixel wants the logical texture at UV (0.3, 0.7). Multiply by the 2048 tiles per side: (614.4, 1433.6). So it's in logical tile (614, 1433), 0.4 and 0.6 of the way across that tile. The indirection texture's pixel for tile (614, 1433) says the tile currently lives in physical slot (12, 40). The physical cache is 64 slots per side, so the shader samples it at ((12 + 0.4) ÷ 64, (40 + 0.6) ÷ 64) = (0.194, 0.634). Next frame the tile may move to another slot; only the indirection pixel changes.

With those three pieces, a shader that wants to sample the logical 128k × 128k texture at UV (u, v) does this:
svt_sample.hlsl · the SVT inner loop

float4 SampleSVT(float2 logicalUv) {
  // 1. Find which logical tile we're in. With a 128k texture and 64-px
  // tiles, there are 2048 tiles on a side. logicalTileCoord is in [0, 2048).
  float2 logicalTileCoord = floor(logicalUv * 2048.0);

  // 2. Read the indirection texture at that coordinate. Each pixel names
  // the physical-cache tile holding the best resident data for this logical
  // tile. When only a coarser mip is resident, every mip-0 entry under that
  // coarse tile points at the same physical tile.
  float4 indirectionEntry = indirection.Load(int3(logicalTileCoord, 0));

  // 3. Translate to physical-cache UV space.
  //    indirectionEntry.xy = which tile slot in the physical cache (in tile units)
  //    indirectionEntry.z  = which mip level is actually resident
  //                          (may be coarser than requested if the finer one is missing)
  // A coarser mip's tile covers a wider span of logical UV, so the
  // within-tile offset must be computed on that mip's tile grid:
  // 2048 tiles per side at mip 0, half as many for each mip above it.
  float  tilesAtResidentMip = 2048.0 / exp2(indirectionEntry.z);
  float2 offsetWithinTile  = frac(logicalUv * tilesAtResidentMip);
  float2 physicalCacheUv   = (indirectionEntry.xy + offsetWithinTile)
                            / physicalCacheTilesPerSide;

  // 4. Sample the physical cache, always at level 0: the cache is a flat
  // tile atlas, not a mip pyramid. The mip decision already happened when
  // the streamer chose which mip's tile to make resident. Bilinear filtering
  // works inside a tile; gutter pixels handle the borders (callout below).
  return physicalCache.SampleLevel(linearSampler, physicalCacheUv, 0);
}

Two texture samples per logical sample. The first one is into a small, cacheable indirection texture; the second is into the physical cache. Hardware bilinear filtering works as normal inside a tile; the borders need gutter pixels (below), and blending between mip levels becomes the engine's job rather than the sampler's.
How the streamer knows what to load

The remaining problem is deciding which tiles of the logical texture should be resident. That depends on the camera, so it isn't known until the frame renders. The usual answer is a feedback pass: a low-resolution render that writes, for every shaded pixel, the logical tile coordinates that pixel would have sampled. After the frame, the CPU (or a compute shader) reads the feedback buffer, deduplicates it, and submits load requests for any tile that's wanted and not yet resident.

Recent GPUs can record this in hardware with Sampler Feedback, a D3D12 feature shipped in Shader Model 6.5[28]. As the shader samples a texture, the hardware records which regions and mip levels it asked for into a separate feedback map, which the engine decodes and reads back. Microsoft's DevBlog demo compares the committed footprint of a tiled, full-mip-chain texturing system under a poor approximation of what to load (524,288 KB, ~512 MiB) and under accurate feedback (51,584 KB, ~50 MiB): about a tenth of the memory. The post itself calls the comparison "a bit silly", but the direction holds[28]. Intel's GDC 2021 demo streams 1,000 objects, each with its own 16k × 16k BC7 texture (350 GB of texture data in total), through a single 1 GB heap with about 230 MB physically resident[29]. On Xbox Series this ships as Sampler Feedback Streaming, part of the Velocity Architecture, which loads only the sub-portions of a mip level the GPU actually needs; Microsoft puts the average gain at about 2.5× effective I/O throughput and memory[8].
Live · Sparse Virtual Texturing
view radius (tiles) 3
physical cache (tile slots) 96
Pan ▶
Pause ⏸
Reset ⟲
tiles wanted
81
tiles resident
0
feedback misses
0
Left is the logical texture with the camera's view square, middle the indirection table, right the physical cache. Press Pan, then try view radius 4 with the 96-slot cache. The feedback requests the tiles in the camera's view square (solid) plus a one-tile prefetch ring (dashed); the streamer uploads 40 tiles a second and evicts the least recently wanted slot. The "feedback misses" counter ticks up once for each tile in view that isn't resident yet. At the default radius 3 the request is 81 tiles, which fit in the 96-slot cache, so after the initial fill the ring hides new tiles before they come into view and misses stop. At radius 4 the request grows to 121 tiles, more than the cache holds, and it thrashes: each upload evicts a tile the view still needs, and misses climb by roughly 25 to 30 a second. Raise the cache to 128 slots and they nearly stop again. This demo streams a single mip level; a real SVT also picks coarser mips as the camera pulls back, which keeps the wanted-tile count tied to screen resolution rather than to how much of the world is in view. Engines fall back to a coarser resident mip while a requested tile streams in, so a miss looks blurry rather than blank.
The hardware: tiled resources and sparse binding

The "physical cache" trick predates GPU hardware support. id Tech 5 transcoded tiles on the CPU and uploaded them as ordinary textures[2]. Current GPUs can manage the cache in hardware as a tiled resource in D3D12 (Microsoft's low-level graphics API)[30] or a sparse image in Vulkan (its cross-platform counterpart)[31]: you allocate a logical-size resource, but its 64 KB tiles are individually backed by physical memory through UpdateTileMappings[32] or vkQueueBindSparse. The shader samples the logical resource directly, with no indirection texture; it only has to avoid tiles that aren't mapped yet (typically by clamping to a resident mip). The GPU's memory-management unit (MMU), the hardware that translates addresses through page tables, does the indirection.

Check Your Understanding Q1 of 2

Why does a sparse virtual texture system have an "indirection texture" in addition to the physical cache?
It stores each tile’s compressed form alongside its coordinates, so the shader can decompress tiles on the fly during sampling instead of waiting for the CPU to decode them at load time.
Logical tile coordinates and physical cache slots don’t match: the same logical tile lives at different physical slots over time, and not every logical tile is resident at any moment. The indirection texture is the per-tile lookup table that translates logical to physical at sample time.
Hardware trilinear filtering needs a side table describing which mip level each physical tile belongs to, or the sampler would blend texels across unrelated mip levels at tile boundaries.
0/2 correct

[Back to File Streaming for Game Engines](index.md)
