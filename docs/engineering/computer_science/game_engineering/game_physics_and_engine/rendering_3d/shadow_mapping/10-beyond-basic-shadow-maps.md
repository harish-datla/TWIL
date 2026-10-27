# Beyond Basic Shadow Maps

- **PCSS / contact hardening:** Search for blockers and scale the PCF kernel by blocker distance, producing sharper contact shadows and softer distant edges.
- **Variance or moment shadow maps:** Store depth moments and filter them like a texture, then estimate visibility with a statistical bound. They support blur, mips, and MSAA but can suffer light bleeding.
- **Ray-traced shadows:** Trace visibility rays instead of sampling a texel grid. This avoids shadow-map acne, aliasing, and peter-panning, though rays need a small origin offset and usually denoising. Hybrid engines combine ray tracing with maps for performance.
