# Shaders → SPIR-V

Vulkan consumes SPIR-V bytecode rather than GLSL source. Compile GLSL offline with `glslc`; HLSL and Slang can also produce SPIR-V. A hardcoded triangle needs no vertex buffer: the vertex shader indexes arrays using `gl_VertexIndex`, and the draw is `vkCmdDraw(cmd, 3, 1, 0, 0)`.

```glsl
#version 450
layout(location = 0) out vec3 fragColor;
vec2 positions[3] = vec2[](vec2(0.0, -0.5), vec2(0.5, 0.5), vec2(-0.5, 0.5));
vec3 colors[3] = vec3[](vec3(1,0,0), vec3(0,1,0), vec3(0,0,1));
void main() {
    gl_Position = vec4(positions[gl_VertexIndex], 0.0, 1.0);
    fragColor = colors[gl_VertexIndex];
}
```

Read `.spv` as 32-bit-aligned words (`ash::util::read_spv` helps in Rust), then create a `VkShaderModule`. Vulkan’s clip space uses a 0..1 depth range and the familiar Y-down framebuffer convention, so projection code must account for the Y orientation. SPIR-V begins with magic number `0x07230203`.
