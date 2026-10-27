# Pitfalls

- **AO on direct light:** It crushes lit contacts. Modulate ambient/indirect light only.
- **Full-sphere SSAO:** It creates gray casts and halos. Use a normal-oriented hemisphere.
- **No range check:** Foreground objects halo distant surfaces. Use a smooth depth-distance weight.
- **Naive box blur:** AO bleeds across silhouettes. Use depth-aware/bilateral blur.
- **“GTAO solves GI”:** It matches an AO reference but remains local and screen-space.
- **“AO simulates indirect light”:** It removes ambient; it never adds bounce or color bleed.
- **Probe light leaking:** Probes interpolate through walls; add visibility data or improve placement.
- **SSGI as full GI:** It misses off-screen contribution and should supplement baked or probe GI.
