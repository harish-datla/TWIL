# Dynamic Rendering

Dynamic rendering (core in Vulkan 1.3) removes the need to create `VkRenderPass` and `VkFramebuffer`. Chain `VkPipelineRenderingCreateInfo` into pipeline creation with the color attachment format, set `renderPass = VK_NULL_HANDLE`, then bracket drawing with `vkCmdBeginRendering` and `vkCmdEndRendering`.

The removed render pass used to perform layout transitions for you. With dynamic rendering, insert them manually every frame:

```text
UNDEFINED → COLOR_ATTACHMENT_OPTIMAL → PRESENT_SRC_KHR
```

The first transition may discard old contents when the frame is cleared. The second waits for color writes so presentation never reads a partially written image. The format in `VkPipelineRenderingCreateInfo` must match the view passed to begin rendering.

```rust
let color_formats = [swapchain_format];
let mut rendering = vk::PipelineRenderingCreateInfo::default()
    .color_attachment_formats(&color_formats);
let pipeline_info = vk::GraphicsPipelineCreateInfo::default()
    .stages(&stages)
    .vertex_input_state(&vertex_input)
    .input_assembly_state(&input_assembly)
    .viewport_state(&viewport_state)
    .rasterization_state(&rasterizer)
    .multisample_state(&multisample)
    .color_blend_state(&color_blend)
    .dynamic_state(&dynamic_state)
    .layout(pipeline_layout)
    .push_next(&mut rendering);
```

On tile-based GPUs, traditional render-pass subpasses—or local reads in newer dynamic-rendering APIs—still matter when one pass should consume another pass’s data without leaving tile memory.
