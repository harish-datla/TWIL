# The Depth Buffer

Create a supported depth image and view, enable depth testing and writing, and attach it to dynamic rendering. Query format support rather than assuming `D32_SFLOAT`, `D32_SFLOAT_S8_UINT`, or `D24_UNORM_S8_UINT` exists.

```text
depth image: DEPTH_STENCIL_ATTACHMENT usage, device-local memory
pipeline: depthTestEnable=true, depthWriteEnable=true
compare: LESS (or GREATER for reversed-Z)
rendering: depthAttachmentFormat + depth attachment
```

Clear depth every frame: clear to 1.0 with LESS, or 0.0 with reversed-Z. `LOAD` retains stale depth and `DONT_CARE` leaves it undefined, so objects can flicker or disappear. Transparent passes commonly use depth test on and depth write off; opaque passes need both enabled.

```rust
let mut rendering = vk::PipelineRenderingCreateInfo::default()
    .color_attachment_formats(&color_formats)
    .depth_attachment_format(depth_format);
let depth_state = vk::PipelineDepthStencilStateCreateInfo::default()
    .depth_test_enable(true).depth_write_enable(true)
    .depth_compare_op(vk::CompareOp::LESS);
let depth_attachment = vk::RenderingAttachmentInfo::default()
    .image_view(depth_image_view)
    .image_layout(vk::ImageLayout::DEPTH_ATTACHMENT_OPTIMAL)
    .load_op(vk::AttachmentLoadOp::CLEAR)
    .store_op(vk::AttachmentStoreOp::DONT_CARE)
    .clear_value(vk::ClearValue { depth_stencil:
        vk::ClearDepthStencilValue { depth: 1.0, stencil: 0 } });
```

Recreate the depth image and view whenever the swapchain extent changes.
