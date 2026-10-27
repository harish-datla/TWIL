# Ambient Occlusion & Global Illumination

Lit scenes can look flat when a wall-floor corner is as bright as open floor. Ambient occlusion darkens ambient light where nearby geometry blocks it. This tutorial builds SSAO on the deferred G-buffer, then surveys techniques that approximate the bounced light AO cannot provide. AO only removes ambient; it is not global illumination.

Time: ~55 min  
Level: Senior  
Prerequisites: [Deferred Rendering](../deferred_rendering_and_g_buffer/index.md) and [PBR](../pbr_materials_and_lighting/index.md).  
Stack: GLSL · C++ & Rust

## What you’ll be able to do

- Define AO and explain why it can only darken ambient light.
- Build SSAO with a normal-oriented kernel, noise, range check, bias, and blur.
- Apply AO only to ambient/indirect lighting.
- Explain what HBAO and GTAO improve and what they still miss.
- Choose a GI family based on acceptable artifacts and cost.

## Sections

- [What AO Is and Is Not](01-what-ao-is-and-is-not.md)
- [SSAO: The Core Idea](02-ssao-the-core-idea.md)
- [SSAO: The Details](03-ssao-the-details.md)
- [Where AO Goes](04-where-ao-goes.md)
- [HBAO and GTAO](05-hbao-and-gtao.md)
- [AO Is Not GI](06-ao-is-not-gi.md)
- [The GI Family Map](07-the-gi-family-map.md)
- [Pitfalls](08-pitfalls.md)
- [What’s Next](09-whats-next.md)
