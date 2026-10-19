# Why Compress

The same week, two complaints land: the download is too big, and loading takes too long. Compression can fix both. A smaller download and package fits storefront and patch budgets. Higher effective load bandwidth comes from reading fewer bytes off disk and inflating them in RAM. Both payoffs come from the same trade of CPU for I/O, and for loading it only pays when the load is I/O-bound and the decoder keeps up with the disk.

## Worked example

A level’s data is 2 GB, and a SATA SSD reads about 0.5 GB/s, so loading it raw takes 4 s. Say it compresses 3:1. The disk reads only 0.67 GB, which takes 1.3 s. One core decodes zstd at about 1.25 GB/s of output, so rebuilding 2 GB takes 1.6 s. If chunks decode while the disk reads the next ones, the load takes about the slower of the two jobs: 1.6 s instead of 4 s. The download is one third the size.

## Two different worlds

General-purpose lossless compression (zstd, LZ4, DEFLATE, and Oodle Kraken) shrinks meshes, audio data, levels, and packages; it is decompressed to RAM before use. GPU texture compression (BCn and ASTC) is lossy, fixed-ratio, and stays compressed in VRAM, where the GPU samples it directly. The general-purpose story comes first; the texture story is in [GPU Texture Compression](08-gpu-texture-compression.md).

Compression does not always speed up loads. On a fast NVMe drive, a slow decoder can become the bottleneck, making compressed loading slower than reading uncompressed data. Decoder speed is covered in [Decode Speed Sets Load Time](06-decode-speed-sets-load-time.md).
