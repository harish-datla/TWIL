# The Anti-Aliasing Problem

Aliasing has distinct causes:

- **Geometric edge aliasing:** pixel-center coverage turns triangle edges into staircases.
- **Shading/specular aliasing:** high-frequency BRDF or normal detail is sampled once per pixel and sparkles.
- **Temporal aliasing:** subpixel features crawl as the camera or object moves.

MSAA targets coverage, normal/roughness prefiltering targets shading frequency, and TAA accumulates different subpixel samples over time. No one technique solves all three. A 0.3-pixel-wide highlight can flash as a pixel-center sample crosses it; extra coverage samples do not fix that shading alias.
