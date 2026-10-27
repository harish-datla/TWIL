# The 3D Vertex

Lighting needs a surface normal and texturing needs UVs, so a vertex becomes position `vec3`, normal `vec3`, and UV `vec2`:

```cpp
struct Vertex { glm::vec3 position; glm::vec3 normal; glm::vec2 uv; };
VkVertexInputBindingDescription binding{0, sizeof(Vertex), VK_VERTEX_INPUT_RATE_VERTEX};
VkVertexInputAttributeDescription attrs[3] = {
    {0, 0, VK_FORMAT_R32G32B32_SFLOAT, offsetof(Vertex, position)},
    {1, 0, VK_FORMAT_R32G32B32_SFLOAT, offsetof(Vertex, normal)},
    {2, 0, VK_FORMAT_R32G32_SFLOAT, offsetof(Vertex, uv)},
};
```

Under non-uniform scale, transform normals with the inverse-transpose of the model’s upper 3×3. Scaling a surface twice along X and transforming its normal directly changes its perpendicular relationship; inverse-transpose preserves it. Lighting consumes these normals in the next module.
