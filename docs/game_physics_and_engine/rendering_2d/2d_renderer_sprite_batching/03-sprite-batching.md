# Sprite Batching

Each frame, clear a CPU-side vertex array and append four vertices per visible sprite. Copy it into a dynamic vertex buffer and issue one indexed draw per batch. Flush when the texture, pipeline, or blend state changes.

The dynamic buffer must be per frame in flight so the CPU never overwrites vertices the GPU is still reading. Within one frame, later flushes append after earlier batches; resetting the buffer offset would make earlier draws read the newest batch. A 16-bit index buffer caps one draw at 65,536 vertices, or 16,384 quads.

```cpp
struct SpriteVertex { float pos[2]; float uv[2]; uint32_t rgba; };

void draw(const Sprite& sprite) {
    if (sprite.texture != batchTexture) flush();
    batchTexture = sprite.texture;
    appendQuad(vertices, sprite);
}

void flush() {
    if (vertices.empty()) return;
    SpriteVertex* dst = mappedVertexBuffer[frameIndex] + frameVertexCount;
    memcpy(dst, vertices.data(), vertices.size() * sizeof(SpriteVertex));
    vkCmdBindDescriptorSets(cmd, VK_PIPELINE_BIND_POINT_GRAPHICS,
        pipelineLayout, 0, 1, &batchTexture.set, 0, nullptr);
    VkDeviceSize offset = 0;
    vkCmdBindVertexBuffers(cmd, 0, 1, &vertexBuffer[frameIndex], &offset);
    vkCmdBindIndexBuffer(cmd, quadIndexBuffer, 0, VK_INDEX_TYPE_UINT16);
    uint32_t count = uint32_t(vertices.size() / 4 * 6);
    vkCmdDrawIndexed(cmd, count, 1, 0, int32_t(frameVertexCount), 0);
    frameVertexCount += uint32_t(vertices.size());
    vertices.clear();
}
```

Production code also needs capacity checks, growth, a 32-bit-index fallback, culling, sort keys, and safe descriptor caching. Host-visible streaming memory is simple but can be slower to read on a discrete GPU.
