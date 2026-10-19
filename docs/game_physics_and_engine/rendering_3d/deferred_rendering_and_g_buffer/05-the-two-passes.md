# The Two Passes

The geometry pipeline lists the G-buffer formats and depth format. Dynamic rendering attaches one image per target, and the fragment shader writes one output per attachment. Attachment order must match shader output locations.

```rust
let color_formats = [albedo_format, normal_format, material_format];
let mut rendering = vk::PipelineRenderingCreateInfo::default()
    .color_attachment_formats(&color_formats)
    .depth_attachment_format(depth_format);
let color_attachments = [gbuf(albedo_view), gbuf(normal_view), gbuf(material_view)];
let info = vk::RenderingInfo::default()
    .color_attachments(&color_attachments)
    .depth_attachment(&depth_attachment);
```

The lighting pass draws one full-screen triangle, reconstructs position, decodes normal, and runs the PBR BRDF per light. Generate the triangle from `gl_VertexIndex`; unlike a quad, it has no shared diagonal that causes duplicate 2×2 helper shading.

Between passes, transition every G-buffer image and depth from attachment-optimal to shader-read-only layouts with a write-to-read barrier. On tile-based GPUs, subpasses or dynamic-rendering local reads can keep the data on-chip and avoid a DRAM round trip.
