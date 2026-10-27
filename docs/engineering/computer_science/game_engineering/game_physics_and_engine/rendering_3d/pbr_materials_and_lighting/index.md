# PBR Materials & Lighting

Your mesh has correct depth, but its material looks right under one light and wrong under the next, metals look like colored plastic, and highlights blow out to white. Physically based rendering addresses these problems with measurable material properties and an energy-aware microfacet model. This tutorial covers the metallic-roughness Cook–Torrance model standardized by glTF, normal maps, punctual lights, HDR tone mapping, and image-based lighting.

Time: ~30 min  
Level: Senior  
Prerequisites: [Going 3D](../perspective_depth_and_meshes/index.md), [Textures, Samplers & Materials](../../rendering_2d/textures_samplers_and_materials/index.md), and 3D Math.  
Stack: GLSL · C++ & Rust

## What you’ll be able to do

- Read the rendering equation and identify what a real-time shader evaluates.
- Implement Cook–Torrance with GGX, Smith, Schlick, and squared roughness.
- Convert glTF base color, metallic, and roughness into BRDF inputs.
- Apply a tangent-space normal map without color-space or green-channel bugs.
- Use punctual lights, HDR output, tone mapping, and image-based lighting.

## Sections

- [What PBR Buys](01-what-pbr-buys.md)
- [The Rendering Equation](02-the-rendering-equation.md)
- [The Microfacet BRDF](03-the-microfacet-brdf.md)
- [Diffuse and Energy Conservation](04-diffuse-and-energy-conservation.md)
- [Metallic-Roughness](05-metallic-roughness.md)
- [Normal Mapping](06-normal-mapping.md)
- [Punctual Lights](07-punctual-lights.md)
- [HDR and Tone Mapping](08-hdr-and-tone-mapping.md)
- [IBL Overview](09-ibl-overview.md)
- [Pitfalls](10-pitfalls.md)
- [What’s Next](11-whats-next.md)
- [Sources and Further Reading](12-sources-and-further-reading.md)
