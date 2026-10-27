# The G-buffer

The lighting pass no longer has mesh or material objects, so the G-buffer stores what lighting needs: base color, normal, metallic/roughness, emissive, and depth. Depth also provides enough information to reconstruct position, so a separate position target is usually redundant.

Vulkan 1.3 guarantees only four color attachments. A portable layout fits in four or fewer plus depth; desktop hardware may report eight, but do not assume it. Keep color and emissive in sRGB formats and normal, metallic, roughness, and occlusion in linear data formats.

One practical layout is RGBA8 base color (4 bytes), RG16F normal (4), RGBA8 material (4), and D32 depth (4): 16 bytes per pixel. At 1080p that is about 33.2 MB written and at least another 33.2 MB read each frame—about 4 GB/s at 60 fps before lighting. At 4K it is roughly 16 GB/s.

A G-buffer stores one surface per pixel. Glass and smoke therefore belong in a later forward pass, not in the deferred targets.
