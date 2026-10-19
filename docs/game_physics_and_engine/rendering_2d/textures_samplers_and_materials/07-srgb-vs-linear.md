# sRGB vs Linear

Image files commonly store color using sRGB encoding, a nonlinear curve that allocates more precision to dark tones. A view with an sRGB format makes the texture unit decode sRGB to linear before filtering, so the shader receives linear values. Lighting, blending, and filtering should happen in linear space. An sRGB swapchain encodes linear shader output back to sRGB on write.

Stored sRGB 128/255 decodes to linear intensity about 0.216, not 0.5. Averaging stored black and white gives 128, but the correct linear average is 0.5, which encodes to about 188. Hardware decoding before filtering produces the correct result.

Do not manually apply `pow(color, 1/2.2)` when the format already performs sRGB conversion, or you encode twice and get washed-out highlights and lifted shadows. Conversely, do not manually decode an already-sRGB-decoded sample. Color/albedo textures use sRGB; normal maps, roughness/metallic, masks, and other data textures use UNORM so their numeric values remain linear.
