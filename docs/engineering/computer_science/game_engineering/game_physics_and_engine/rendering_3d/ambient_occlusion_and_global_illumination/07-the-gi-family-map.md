# The GI Family Map

Choose a GI technique by choosing the failure mode and cost you can accept:

| Family | What it does | Main failure or cost |
| --- | --- | --- |
| Baked lightmaps | Offline radiosity/path tracing into textures | Static under moving lights/geometry |
| Probes | Sample indirect light at points, often SH | Leaks through walls; low-frequency |
| DDGI | Update probes with rays each frame | Diffuse-only; hardware RT often needed; leaks remain |
| Voxel cone tracing | Voxelize and cone-trace a mip pyramid | Approximate, memory-heavy, leaks through thin geometry |
| SSGI | Gather indirect light from screen pixels | Off-screen blind; view-dependent popping |
| Ray-traced/SDF hybrids | Trace scene visibility with RT or distance fields | Noise and denoising cost; approximations remain |

Baked lighting is cheap at runtime but cannot react to dynamic changes. Probes interpolate and can blend outdoor light through a wall. DDGI adds visibility tests but is still an approximation. SSGI is a supplement, not full GI. Modern systems such as Lumen combine screen traces, signed-distance fields, surface caches, and optional hardware ray tracing.

A probe grid with 32×16×32 probes and nine SH coefficients per color channel uses roughly 1.8 MB in float32, but interpolation between probes on opposite sides of a wall can leak light.
