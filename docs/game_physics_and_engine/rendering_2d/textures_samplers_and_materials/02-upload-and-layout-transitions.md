# Upload and Layout Transitions

Optimal-tiled images are generally not host-visible, so upload decoded pixels through a host-visible staging buffer, then issue `vkCmdCopyBufferToImage`. The copy needs two transitions:

```text
UNDEFINED → TRANSFER_DST_OPTIMAL
copy buffer to image
TRANSFER_DST_OPTIMAL → SHADER_READ_ONLY_OPTIMAL
```

The first transition intentionally discards old contents. The second makes transfer writes visible to shader reads. Sampling an image still in `TRANSFER_DST_OPTIMAL` or `UNDEFINED` is the classic validation error. `GENERAL` is legal but can be slower.

For a 512 × 256 RGBA8 image, the staging buffer is `512 × 256 × 4 = 524,288` bytes. With `bufferRowLength = 0`, rows are tightly packed at 2,048 bytes. If the loader pads rows to 4,096 bytes, set `bufferRowLength = 1024` because the field counts texels.

```cpp
VkImageCreateInfo info{ VK_STRUCTURE_TYPE_IMAGE_CREATE_INFO };
info.imageType = VK_IMAGE_TYPE_2D;
info.format = VK_FORMAT_R8G8B8A8_SRGB;
info.extent = { width, height, 1 };
info.mipLevels = 1; info.arrayLayers = 1;
info.tiling = VK_IMAGE_TILING_OPTIMAL;
info.usage = VK_IMAGE_USAGE_TRANSFER_DST_BIT | VK_IMAGE_USAGE_SAMPLED_BIT;
info.initialLayout = VK_IMAGE_LAYOUT_UNDEFINED;

VkImageMemoryBarrier barrier{ VK_STRUCTURE_TYPE_IMAGE_MEMORY_BARRIER };
barrier.image = image;
barrier.subresourceRange = { VK_IMAGE_ASPECT_COLOR_BIT, 0, 1, 0, 1 };
barrier.oldLayout = VK_IMAGE_LAYOUT_UNDEFINED;
barrier.newLayout = VK_IMAGE_LAYOUT_TRANSFER_DST_OPTIMAL;
barrier.srcAccessMask = 0;
barrier.dstAccessMask = VK_ACCESS_TRANSFER_WRITE_BIT;
vkCmdPipelineBarrier(cmd, VK_PIPELINE_STAGE_TOP_OF_PIPE_BIT,
    VK_PIPELINE_STAGE_TRANSFER_BIT, 0, 0, nullptr, 0, nullptr, 1, &barrier);

VkBufferImageCopy region{};
region.imageSubresource = { VK_IMAGE_ASPECT_COLOR_BIT, 0, 0, 1 };
region.imageExtent = { width, height, 1 };
vkCmdCopyBufferToImage(cmd, stagingBuffer, image,
    VK_IMAGE_LAYOUT_TRANSFER_DST_OPTIMAL, 1, &region);

barrier.oldLayout = VK_IMAGE_LAYOUT_TRANSFER_DST_OPTIMAL;
barrier.newLayout = VK_IMAGE_LAYOUT_SHADER_READ_ONLY_OPTIMAL;
barrier.srcAccessMask = VK_ACCESS_TRANSFER_WRITE_BIT;
barrier.dstAccessMask = VK_ACCESS_SHADER_READ_BIT;
vkCmdPipelineBarrier(cmd, VK_PIPELINE_STAGE_TRANSFER_BIT,
    VK_PIPELINE_STAGE_FRAGMENT_SHADER_BIT, 0, 0, nullptr, 0, nullptr, 1, &barrier);
```

This simplified upload uses a one-shot command buffer and `vkQueueWaitIdle`. Production code batches uploads and may use a transfer queue, which additionally needs queue-family ownership transfers. Vulkan 1.3 synchronization-2 expresses stages and access masks in the barrier structs.
