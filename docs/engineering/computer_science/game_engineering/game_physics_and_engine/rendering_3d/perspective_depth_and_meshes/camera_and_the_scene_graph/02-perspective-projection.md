# Perspective Projection

For a camera looking down −Z, vertical FOV, aspect, and focal factor `f = 1/tan(FOV/2)`, a view-space point projects to:

```text
x_ndc = (f / aspect) × x / (−z)
y_ndc = f × y / (−z)
```

A 60° FOV gives `f=1.732`. Point `(2,1,-10)` in a 16:9 view becomes approximately `(0.195,0.173)`; at −20 it becomes half as far from center.

Vulkan’s NDC is right-handed, its depth range is 0..1, and its framebuffer is Y-down. These are separate facts. `GLM_FORCE_DEPTH_ZERO_TO_ONE` fixes depth range but not Y; still flip `projection[1][1]`, or use a negative viewport height. In Rust, `glam::Mat4::perspective_rh` targets 0..1; `perspective_rh_gl` targets OpenGL’s −1..1.

```cpp
glm::mat4 projection = glm::perspective(glm::radians(60.0f), aspect, 0.1f, 1000.0f);
projection[1][1] *= -1.0f;
glm::mat4 view = glm::lookAt(cameraPos, cameraTarget, glm::vec3(0,1,0));
glm::mat4 clipFromModel = projection * view * model;
```

Reversed-Z maps near to 1 and far to 0, uses a floating-point depth buffer, compares with GREATER, and clears to 0. It improves far precision by complementing the 1/Z distribution; it is an additional swap on top of Vulkan’s 0..1 range and does not help a UNORM buffer.
