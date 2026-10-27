# World → Render

The world is authoritative state. A read-only render pass walks it and emits a render description—position, atlas UV, tint, and layer—which the sprite batcher turns into draw calls. The simulation never calls Vulkan, so it can run headlessly and the renderer can interpolate or move to another thread later.

```cpp
void buildBatch(const World& world, float alpha, SpriteBatch& out) {
    for (EntityId id : world.withSprites()) {
        const Transform& prev = world.previous(id);
        const Transform& cur = world.current(id);
        float x = std::lerp(prev.x, cur.x, alpha);
        float y = std::lerp(prev.y, cur.y, alpha);
        const Sprite& sprite = world.sprite(id);
        out.quad(x, y, cur.rot, sprite.atlasRect, sprite.tint, sprite.layer);
    }
}
```

For 51 sprites, this emits 51 quads, or 204 20-byte vertices—4,080 bytes—without changing the world. Running the pass twice produces the same description, which makes it safe to test, repeat, skip, or move.
