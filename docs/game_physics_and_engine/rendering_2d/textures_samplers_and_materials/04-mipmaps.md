# Mipmaps

A distant texel can cover dozens of source texels. Sampling one source texel makes a surface shimmer as the camera moves. Mipmaps are prefiltered half-size copies that reduce minification aliasing. The hardware chooses a level using texture-coordinate derivatives from the 2×2 quad, then filters within and between levels.

The full chain has:

```text
floor(log₂(max(width, height))) + 1 levels
```

A 1024 × 512 texture has 11 levels, ending at 1 × 1. In RGBA8 the base level is 2,097,152 bytes and the full chain is about 2,796,204 bytes—roughly one third more, because each level is a quarter the size of its predecessor.

Production assets usually ship with mips baked into KTX2 or DDS files. Runtime `vkCmdBlitImage` generation requires optimal-tiling support for `BLIT_SRC`, `BLIT_DST`, and linear filtering, queried with `vkGetPhysicalDeviceFormatProperties`. The image also needs `TRANSFER_SRC` usage and a graphics-capable queue. BCn textures generally cannot be blit destinations because a blit cannot encode compressed blocks, so their mip chains are precomputed.
