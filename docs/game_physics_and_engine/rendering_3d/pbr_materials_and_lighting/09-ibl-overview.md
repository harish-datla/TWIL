# IBL Overview

Punctual lights cannot make a chrome sphere reflect an orange sunset. Image-based lighting supplies environment light. Diffuse IBL uses an irradiance map: the environment convolved with the cosine lobe. Specular IBL commonly uses a split sum:

- a prefiltered environment cubemap, blurred per roughness in its mip levels;
- a 2D BRDF integration LUT indexed by `NdotV` and roughness.

The shader combines the prefiltered environment with `F0 × scale + bias` from the LUT. A 256×256 cubemap has nine mip levels. With a simple linear mapping, roughness 0.5 samples mip 4, the 16×16 level.

Capture, importance-sampled prefiltering, and LUT generation are substantial topics of their own. The split-sum approximation assumes view, normal, and reflection relationships that become less accurate at grazing angles. Ambient occlusion can darken crevices, but it is not global illumination and does not produce bounced light.
