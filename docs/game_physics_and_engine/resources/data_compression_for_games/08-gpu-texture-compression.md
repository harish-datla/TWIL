# GPU Texture Compression

A 2048 × 2048 RGBA texture occupies 16 MiB in VRAM. zstd could shrink it on disk, but the GPU cannot sample a zstd stream; after decoding, it would occupy 16 MiB again. BCn (BC1–BC7) and ASTC are lossy, fixed-rate, block-based formats. The texture stays compressed in VRAM, and texture hardware decodes only the block being sampled.

Fixed rate and block layout enable random access. For BCn, blocks are 4×4 texels. A variable-rate stream such as DEFLATE cannot find an arbitrary texel in constant time, so it is not a runtime texture format.

## Worked example

BC1 stores each 4×4 block in 8 bytes; BC7 stores it in 16 bytes. A 2048 × 2048 texture has `512 × 512` blocks:

- BC1: 2 MiB, an 8:1 reduction against RGBA.
- BC7: 4 MiB, a 4:1 reduction against RGBA.

For texel `(1000, 517)` in BC7, divide by 4 to get block `(250, 129)`. In a simple row-major layout:

```text
block index = 129 × 512 + 250 = 66,298
byte offset = 66,298 × 16 = 1,060,768
```

Real GPUs rearrange blocks for locality, but the address remains directly computable. In a zstd stream, the corresponding bytes can be located only by decoding from the stream’s beginning or from an independent block.

| Property | General-purpose (zstd/Kraken) | GPU texture (BCn/ASTC) |
| --- | --- | --- |
| Decoded | Fully to RAM before use | Per block in the sampler |
| Ratio | Variable, content-dependent | Fixed by the format |
| Access | Sequential or block-seekable | Random by texel |
| Lossy? | No | Yes |

These forms compose: ship BCn textures and compress the package with a lossless codec for download size. A BCn blob can be uploaded largely as-is, while a zstd-compressed asset must first be decoded into a separate buffer and cannot be memory-mapped and used directly.

## Check your understanding

Loads are slow. Profiling shows the NVMe finishes reading the compressed package almost instantly, but the frame stalls. What is the likely fix?

**Answer:** Switch to a faster-decoding codec, such as LZ4 or a lower zstd level, even if the ratio is worse.
