# Pitfalls

- **“Batching speeds up the GPU”:** It primarily cuts CPU and driver draw overhead.
- **Draws remain high:** Texture changes break batches; use an atlas or array.
- **Overwriting an in-flight vertex buffer:** Keep dynamic buffers per frame in flight.
- **Atlas edge seams:** Add a half-texel inset, padding gutter, and edge extrusion.
- **Dark cutout halos:** Use premultiplied alpha under filtering and mipmaps.
- **Transparent sprites show through incorrectly:** Sort back-to-front and disable depth writes.
- **Pushing three matrices:** A 192-byte MVP exceeds the 128-byte minimum; push the 64-byte view-projection.
- **The 16k-quad limit:** Use 32-bit indices or split the batch.
