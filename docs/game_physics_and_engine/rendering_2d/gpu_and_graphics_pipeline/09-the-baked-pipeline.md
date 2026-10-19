# The Baked Pipeline

A hitch the first time a material appears often comes from compiling GPU code mid-frame. Explicit APIs bake shader stages and fixed-function state into a pipeline object that can be optimized as a whole and bound efficiently.

The baked state includes shaders, rasterizer settings, depth comparison, blending, and viewport state. Modern Vulkan adds dynamic state and shader objects, so not every value must be immutable; this is the explicit-API model, not a universal rule.

If an engine has 10 shader pairs, 3 blend modes, 2 depth setups, and 2 cull modes, baking every combination produces:

```text
10 × 3 × 2 × 2 = 120 pipeline objects
```

Compile pipelines ahead of time or cache them on disk. Making a state dynamic reduces combinatorial growth, at the cost of some flexibility or runtime work. The next tutorial turns this model into a `VkPipeline`, connects it to a swapchain, and draws a real triangle.
