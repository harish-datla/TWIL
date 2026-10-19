# The Quad and Camera

Every sprite is a quad made from two triangles, four vertices, and six indices `(0,1,2, 2,3,0)`. Each vertex carries world position, UV, and tint. An orthographic camera maps world units to the screen without perspective scaling.

For a pixel-space orthographic projection with width `W` and height `H`:

```text
x_ndc = 2x / W − 1
y_ndc = 2y / H − 1
```

The view is a pan and zoom; compute `viewProjection = orthographic × view` once per frame and apply it in the vertex shader. Keep sprite positions in world space so moving the camera does not rebuild geometry.

With a 1280 × 720 screen, camera centered at `(640,360)`, and 2× zoom, world point `(700,400)` becomes screen `(760,440)` and NDC `(0.19, 0.22)`. Vulkan’s Y-down framebuffer mapping is not the same thing as world handedness. A +Y-up game needs a flip by reversing the orthographic top/bottom or using a negative viewport height. Zoom around the screen or cursor rather than the world origin to avoid drift.
