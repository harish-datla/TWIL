# Pitfalls

- **Sampling the wrong layout:** Add the `TRANSFER_DST_OPTIMAL → SHADER_READ_ONLY_OPTIMAL` barrier before sampling.
- **Washed-out color:** Do not double-encode with an sRGB swapchain plus manual `pow(1/2.2)`.
- **Wrong normal map:** Data textures must use UNORM, not sRGB.
- **Anisotropy errors:** Enable the feature and clamp to `maxSamplerAnisotropy`.
- **Mip bands:** Trilinear needs both `minFilter = LINEAR` and `mipmapMode = LINEAR`.
- **Silent descriptor-pool failure:** Size pools per descriptor type and set count.
- **Updating an in-use set:** Update before use or keep one set per frame in flight.
- **One `vkAllocateMemory` per texture:** Use VMA or another sub-allocator.
