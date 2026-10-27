# Immediate-Mode vs Tile-Based

Desktop and mobile GPUs organize framebuffer traffic differently.

- **Immediate-mode rendering (IMR):** Common on desktop NVIDIA and AMD. Primitives are shaded in submission order against a framebuffer in main memory, with caches and compression reducing traffic. Desktop GPUs may walk screen tiles internally, but do not generally hold an entire render pass on-chip as a mobile tiler does.
- **Tile-based rendering (TBR/TBDR):** Common on Arm Mali, Imagination PowerVR, Apple, and Adreno. The GPU bins geometry into screen tiles, shades one tile in fast on-chip memory, and stores only the requested results. Tile-based deferred renderers can resolve visibility before shading hidden fragments.

A 16×16 tile with 4-byte color and 4-byte depth needs `256 × 8 = 2,048` bytes on-chip. A full 1080p frame needs about 16.6 MB at the same 8 bytes per pixel. Splitting a pass so the second pass samples the first can force a 16.6 MB store and reload—about 33 MB of extra traffic per frame, roughly 2 GB/s at 60 fps.

On mobile, use Vulkan subpasses and input attachments—or local reads where supported—to keep intermediate data on tile. Use load/store operations that discard attachments when their old or final contents are unnecessary. Tile-based deferred rendering is not the same thing as deferred shading, which uses a G-buffer.
