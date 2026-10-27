# Sampling in the Shader

One shader call performs filtering, mip selection, and addressing:

```glsl
#version 450
layout(location = 0) in vec2 fragTexCoord;
layout(set = 0, binding = 1) uniform sampler2D texSampler;
layout(location = 0) out vec4 outColor;
void main() {
    outColor = texture(texSampler, fragTexCoord);
}
```

`binding = 1` must match the descriptor-set layout and `set = 0` must match the set bound at draw. GLSL compiled with `glslc` is valid SPIR-V input alongside Slang and HLSL workflows.

On a 1024 × 512 texture, UV `(0.25, 0.75)` addresses around `(256, 384)`. With `REPEAT`, `(1.25, 0.75)` wraps to the same location; with `CLAMP_TO_EDGE`, it reaches the last column instead. Avoid sampling inside divergent control flow when relying on implicit derivatives; use `textureLod` or `textureGrad` when explicit gradients are needed.
