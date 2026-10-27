# Pitfalls

- **“Vulkan is left-handed”:** It is right-handed; Y-down framebuffer mapping and 0..1 depth range are separate issues.
- **Assuming the depth macro fixes Y:** `GLM_FORCE_DEPTH_ZERO_TO_ONE` changes depth range only; flip projection Y separately.
- **Forgetting to clear depth:** Use `loadOp = CLEAR` every frame.
- **Inside-out model:** Make `frontFace` agree with framebuffer winding after the Y flip.
- **Garbled UBO transforms:** Match std140 alignment; vec3 aligns to 16 bytes.
- **Using reversed-Z with UNORM:** The precision benefit requires a floating-point depth buffer.
- **Assuming a depth format:** Query format support on the selected physical device.
- **Opaque objects do not occlude:** Enable both depth test and depth write.
