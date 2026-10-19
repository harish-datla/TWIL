# The Graphics Pipeline

`VkPipeline` freezes shader stages, input assembly, rasterization, multisampling, depth/stencil, color blending, and the pipeline layout. The layout is a separate interface describing descriptors and push constants visible to shaders; it is not the pipeline itself.

Make viewport and scissor dynamic with `VK_DYNAMIC_STATE_VIEWPORT` and `VK_DYNAMIC_STATE_SCISSOR`. Set them while recording commands so a window resize does not require a new pipeline. The pipeline still contains the counts and state descriptions, while the actual rectangles are set per command buffer.
