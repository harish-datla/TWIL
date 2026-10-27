# Model → View → Projection


Your first Vulkan triangle renders upside down, or vanishes when the camera gets close. Both come from the last step of the chain. Three matrices get a model onto the screen. Model takes object space to world space. View takes world to eye space, and is the inverse of the camera's world transform. Projection takes eye space to clip space, a homogeneous space, where each point carries a fourth number w and the divide by it hasn't happened yet.

After the vertex shader writes clip-space (x, y, z, w), the GPU does the perspective divide (xyz / w) to get NDC, then the viewport transform to get framebuffer pixels. The projection matrix is where the API's clip-space conventions live, and this is where OpenGL and Vulkan differ in ways that produce upside-down or depth-broken renders.

Worked example

The vertex shader writes clip (2, 1, 0.5, 4). The divide gives NDC (2/4, 1/4, 0.5/4) = (0.5, 0.25, 0.125): right of center, a little above the middle in OpenGL's Y-up NDC, and near the front of Vulkan's 0…1 depth range.

With a 60° vertical field of view, the projection's y scale is 1 / tan 30° = 1.73. A point 1 unit up and 5 units in front of the camera lands at NDC y = 1.73 / 5 = 0.35. In OpenGL's Y-up NDC that's above center, where it belongs; in Vulkan's Y-down NDC it's below center, so the matrix negates that 1.73, which is what the listing below does.
OpenGLVulkan
NDC Y directionupdown
Clip depth range−w ≤ z ≤ w  (NDC −1…1)0 ≤ z ≤ w  (NDC 0…1)
NDC handednessleft-handedright-handed

Sources: the Vulkan specification's vertex post-processing chapter and Wellings' coordinate-system writeup[5][6].

Two consequences for the projection matrix:

    The [0, 1] depth range changes the matrix's third row, which is why libraries ship distinct variants: glm gates it with GLM_FORCE_DEPTH_ZERO_TO_ONE (and exposes perspectiveRH_ZO vs perspectiveRH_NO)[3]; glam splits Mat4::perspective_rh (zero-to-one, for Vulkan, WebGPU, Direct3D) from perspective_rh_gl (negative-one-to-one, for OpenGL)[4].
    The Y-down NDC is handled one of a few ways: negate gl_Position.y in the shader, bake a Y-flip into the projection, or set a negative VkViewport.height, which negates clip-space Y in the viewport transform. Negative viewport height came with VK_KHR_maintenance1 and is core since Vulkan 1.1[16].

Trap: "Vulkan is left-handed" is wrong, and it hides three separate things

Vulkan's NDC is right-handed. The visible flip people hit is the Y-down framebuffer mapping, which is separate from the depth range, which is separate again from your world-space handedness[5]. Conflating the three is a common source of OpenGL→Vulkan porting bugs. And reversed-Z, which maps the near plane to depth 1 and the far plane to 0, is yet another, orthogonal, precision technique; it is not the same as the [0, 1] range, though it pays off only with a [0, 1] range and a floating-point depth buffer.

Worked example

A Vulkan projection with near 0.1 and far 1,000 maps a point at distance d to depth 1000 (d − 0.1) / (999.9 d). At d = 0.2 that's 0.50; at d = 1, 0.90; at d = 10, 0.99. Ninety-nine percent of the depth range covers the first 10 units of a 1,000-unit view, which is why distant surfaces z-fight and why reversed-Z helps.

Feed Vulkan an OpenGL projection with the same planes instead, and depth reaches 0 only at d = 2 × 1000 × 0.1 / 1000.1 ≈ 0.2. Anything closer comes out below 0, outside Vulkan's 0…1 range, and is clipped.

The widget renders the same scene for a chosen target API. Untick the box to feed it a projection written for the other API, or mismatch the import handedness; the readout names each thing that breaks:
Vulkan (Y-down, 0…1)
OpenGL (Y-up, −1…1)
projection written for this API
mismatch import handedness
The F and the two triangles are the scene as the chosen API draws it, and the panel lists the target's conventions and everything that broke. Start with both boxes as they load, then untick the projection box, then tick the handedness box. Feed Vulkan a projection written for OpenGL and the scene renders upside down; OpenGL's −1…1 depth also lands in Vulkan's 0…1 clip volume, so geometry closer than about twice the near distance (the red triangle) is clipped. Feed OpenGL a Vulkan projection and it is upside down too, with half the depth range unused. Mismatch import handedness and the scene mirrors along X while every triangle's winding reverses, so back-face culling throws away the faces that should be visible (dashed outlines). Y direction, depth range, and world handedness are independent; the readout lists each one that broke.
A Vulkan-ready perspective matrix
C++
Rust

// Define this BEFORE including glm so depth maps to [0, 1] for Vulkan:
//   #define GLM_FORCE_DEPTH_ZERO_TO_ONE
glm::mat4 proj = glm::perspective(glm::radians(60.0f), aspect, 0.1f, 1000.0f);
proj[1][1] *= -1.0f;  // flip Y: glm assumes GL's Y-up NDC, Vulkan's is Y-down
// Alternative to the flip: set a negative VkViewport.height.


[Back to 3D Math for Games](index.md)
