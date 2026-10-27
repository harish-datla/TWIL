# Alpha Blending

Transparency combines new source color with the framebuffer destination:

```text
result = source × sourceFactor + destination × destinationFactor
```

Straight alpha stores unmultiplied color and uses `SRC_ALPHA, ONE_MINUS_SRC_ALPHA`. Premultiplied alpha stores RGB already multiplied by alpha and uses `ONE, ONE_MINUS_SRC_ALPHA`.

Premultiplied alpha avoids dark halos. Filtering a transparent texel with arbitrary RGB can pull that RGB into a neighboring partially covered sample. With premultiplied texels, transparent RGB is zero and interpolation remains correct through filtering and mipmaps.

```cpp
VkPipelineColorBlendAttachmentState blend{};
blend.blendEnable = VK_TRUE;
blend.srcColorBlendFactor = VK_BLEND_FACTOR_ONE;
blend.dstColorBlendFactor = VK_BLEND_FACTOR_ONE_MINUS_SRC_ALPHA;
blend.colorBlendOp = VK_BLEND_OP_ADD;
blend.srcAlphaBlendFactor = VK_BLEND_FACTOR_ONE;
blend.dstAlphaBlendFactor = VK_BLEND_FACTOR_ONE_MINUS_SRC_ALPHA;
blend.alphaBlendOp = VK_BLEND_OP_ADD;
blend.colorWriteMask = VK_COLOR_COMPONENT_R_BIT | VK_COLOR_COMPONENT_G_BIT |
                       VK_COLOR_COMPONENT_B_BIT | VK_COLOR_COMPONENT_A_BIT;
```

For straight-alpha assets, use `SRC_ALPHA` as the source color factor instead.
