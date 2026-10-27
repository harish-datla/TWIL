# Assembling the render frame

Each pass reads what an earlier pass wrote: lighting needs the G-buffer, and the shadow test needs the shadow map. Run bloom before TAA and bright edge pixels that flicker from frame to frame, which TAA would have settled, can get spread into a shimmering glow. renderFrame reads the interpolated, read-only snapshot and runs the deferred passes in order:

    Shadow depth pass from the light (Shadows).
    G-buffer geometry pass (Deferred): static meshes and skinned meshes via vertex-shader skinning from the per-frame matrix palette (Skeletal Animation), writing gAlbedo/gNormal/gMaterial.
    SSAO (AO/GI).
    Full-screen lighting: the Deferred module's per-light loop calling cookTorrance (the PBR BRDF), with shadowFactor + PCF folded in.
    Forward pass for transparency (a G-buffer holds one opaque surface per pixel, so blended transparents get their own forward pass).
    Post: TAA resolve on the HDR image, then bloom pyramid → tonemap (Post-Processing). TAA commonly runs before the rest of post, on the HDR image, so bloom and tonemap see the resolved, stable frame[3].
    ImGui overlay → present.

The only new content here is the ordering. Everything else is a call into a module you already built, reading transforms that were lerp/slerp-interpolated and mutating nothing.
