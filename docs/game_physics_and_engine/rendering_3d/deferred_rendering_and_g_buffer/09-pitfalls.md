# Pitfalls

- **“Deferred is just faster”:** It trades bandwidth for reduced redundant lighting.
- **Storing position:** Reconstruct it from depth; Vulkan Z is already 0..1.
- **Assuming eight color attachments:** Vulkan 1.3 guarantees four; query or fit within four.
- **Using sRGB for data:** Normals, metallic, and roughness are linear; base color and emissive are sRGB.
- **Putting transparency in the G-buffer:** It stores one surface; draw translucents forward afterward.
- **Forgetting MSAA cost:** Deferred MSAA needs per-sample targets or edge-specific shading.
- **Conflating clustered and deferred:** Light culling is independent of the shading architecture.
- **Uniform depth slices:** Use exponential slices so cluster shapes remain useful across distance.
