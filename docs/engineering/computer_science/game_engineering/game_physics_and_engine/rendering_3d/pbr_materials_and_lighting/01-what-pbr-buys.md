# What PBR Buys

Artists often tune a shiny exponent until a crate looks right, then retune it under another light. Physically based shading instead describes surfaces with energy conservation, Fresnel reflectance, and a statistical microfacet distribution. A material authored with base color, metallic, and roughness remains plausible across lighting setups.

“Physically based” means plausible under the model’s approximations, not physically exact. PBR is a family of parameterizations and BRDF choices. This tutorial uses glTF 2.0’s metallic-roughness Cook–Torrance model with GGX, Smith masking, and Schlick Fresnel.

Roughness is perceptual. The BRDF uses `alpha = roughness²`; skipping the square makes mid-roughness highlights too broad and collapses the useful range of the artist’s slider.
