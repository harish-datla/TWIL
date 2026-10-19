# Per-Object Data

A scene with 1,000 props needs 1,000 model matrices per frame. The choices are:

- A push constant for one object or one premultiplied MVP.
- A uniform buffer, usually with std140 layout.
- A dynamic uniform buffer with per-object aligned offsets.
- An SSBO using std430, indexed by instance or draw ID; common in GPU-driven renderers.

Std140 is a frequent silent bug. A `vec3` aligns to 16 bytes even though its data occupies 12; matrices are 64 bytes and array/struct members round up to 16. A block `{ float intensity; vec3 color; }` places intensity at 0 and color at 16 and occupies 32 bytes, while a naïve C struct puts color at offset 4. Pad explicitly or avoid vec3 in uniform blocks.

Dynamic UBO offsets must be multiples of `minUniformBufferOffsetAlignment`; query it rather than assuming. With 1,000 64-byte matrices and 256-byte alignment, a dynamic UBO consumes 256,000 bytes instead of the packed SSBO’s 64,000.
