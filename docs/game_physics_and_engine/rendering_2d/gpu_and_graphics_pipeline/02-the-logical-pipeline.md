# The Logical Pipeline

In the classic graphics pipeline, a triangle flows through:

```text
vertex shader → clip / cull → rasterize → fragment shader
             → depth / stencil → blend → framebuffer
```

The vertex shader runs once per vertex and outputs a clip-space position. After clipping, the GPU performs the perspective divide (`x / w`, `y / w`, `z / w`) to produce NDC. The viewport transform maps NDC to framebuffer pixels. Vulkan’s clip volume uses `0 ≤ zc ≤ wc`, giving a depth range of [0, 1]; a negative viewport height performs the usual Y flip.

For a 90° vertical field of view, 16:9 aspect, near 0.3, far 100, and an 800 × 450 viewport, object position `(1, 0.5, -3)` can become clip `(0.5625, 0.5, 2.71, 3)`, NDC `(0.19, 0.17, 0.90)`, and framebuffer position approximately `(475, 187)` after the Y flip.

This is an as-if contract. Real hardware reorders work for speed, such as using early depth tests, but remains functionally equivalent. A fragment is a candidate pixel that depth, stencil, and blending may still reject. Tessellation and geometry stages are optional; a fragment shader can also be omitted for depth-only passes.
