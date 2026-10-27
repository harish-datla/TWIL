# Textures, Samplers & Materials

You load a PNG onto your triangle and it renders black, shimmers as it recedes, or appears washed out and too bright. Each symptom has a specific Vulkan cause. This tutorial builds the image, view, sampler, staging upload, descriptor, mip, and sRGB pieces in C++ and Rust (ash).

Time: ~55 min  
Level: Senior  
Prerequisites: [Your First Triangle in Vulkan](../first_triangle_in_vulkan/index.md) and [The GPU & the Graphics Pipeline](../gpu_and_graphics_pipeline/index.md).  
Stack: C++ (Vulkan) · Rust (ash)

## What you’ll be able to do

- Upload pixels through a staging buffer and place both image-layout barriers correctly.
- Configure nearest, bilinear, trilinear, and anisotropic filtering within device limits.
- Calculate mip-chain size and choose offline baking versus runtime generation.
- Connect an image and sampler to a shader with a correctly sized descriptor pool.
- Keep color textures, data textures, and the swapchain in the right color spaces.

## Sections

- [A Texture Is a VkImage](01-a-texture-is-a-vkimage.md)
- [Upload and Layout Transitions](02-upload-and-layout-transitions.md)
- [Samplers](03-samplers.md)
- [Mipmaps](04-mipmaps.md)
- [Sampling in the Shader](05-sampling-in-the-shader.md)
- [Descriptor Sets](06-descriptor-sets.md)
- [sRGB vs Linear](07-srgb-vs-linear.md)
- [Bindless](08-bindless.md)
- [Pitfalls](09-pitfalls.md)
- [What’s Next](10-whats-next.md)
