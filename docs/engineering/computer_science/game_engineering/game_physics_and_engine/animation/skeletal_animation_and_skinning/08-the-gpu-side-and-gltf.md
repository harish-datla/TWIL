# The GPU side & glTF

The CPU knows where every joint is this frame, and the vertices live on the GPU, so the joint data has to cross over every frame. Each frame the CPU builds the matrix palette (one skinning matrix per joint) and uploads it; the vertex shader reads the per-vertex joint indices and weights, sums the four palette matrices, and transforms position and normal. Here skinning happens in the vertex shader; some engines skin in a compute pass instead and reuse the skinned vertices across render passes.

Worked example

A 100-joint character's palette is 100 × 64 bytes = 6,400 bytes, comfortably inside a 16 KiB uniform buffer, which holds 256 matrices. A thousand such characters in one storage buffer is 6.4 MB to rebuild and upload every frame, 384 MB/s at 60 fps, on top of the CPU time to compute every pose.
UBO caps push large or instanced palettes to an SSBO

Vulkan's guaranteed-minimum maxUniformBufferRange is 16 KiB through Vulkan 1.3 (1.4 raises it to 64 KiB), and a mat4 is 64 bytes, so a portable UBO (uniform buffer, a small read-only constant buffer) palette holds 256 matrices[11]. Fine for one ~100-joint skeleton, but many skinned instances in one buffer need an SSBO (storage buffer, guaranteed up to 128 MiB), indexed by instanceID · jointCount + joint. This is the same UBO-vs-SSBO trade as in Going 3D. At crowd scale the per-character CPU work of building all those palettes is what VAT removes.
Vertex-shader skinning (GLSL 4.5, SSBO palette)
GLSL

#version 450

layout(location = 0) in vec3  in_position;
layout(location = 1) in vec3  in_normal;
layout(location = 2) in uvec4 in_jointIndices;   // glTF JOINTS_0: unsigned, so a *_UINT vertex format
layout(location = 3) in vec4  in_jointWeights;   // glTF WEIGHTS_0 (sum to 1)

layout(std430, set = 0, binding = 0) readonly buffer Palette {
    mat4 skinningMatrix[];      // globalPose[j] * inverseBind[j], one per joint
} palette;

layout(std140, set = 0, binding = 1) uniform Camera {
    mat4 viewProj;              // assumes the joint globals include the character's world transform
} camera;

layout(location = 0) out vec3 out_normal;

void main() {
    mat4 skin = in_jointWeights.x * palette.skinningMatrix[in_jointIndices.x]
              + in_jointWeights.y * palette.skinningMatrix[in_jointIndices.y]
              + in_jointWeights.z * palette.skinningMatrix[in_jointIndices.z]
              + in_jointWeights.w * palette.skinningMatrix[in_jointIndices.w];  // LBS
    vec4 skinnedPos = skin * vec4(in_position, 1.0);
    vec3 skinnedNrm = normalize(mat3(skin) * in_normal);   // rigid approx; invT under non-uniform scale
    gl_Position = camera.viewProj * skinnedPos;
    out_normal  = skinnedNrm;
}

glTF skins, concretely

A glTF skin has a joints array (node indices), an inverseBindMatrices accessor (one mat4 per joint, same order), and an optional skeleton root hint (it doesn't change the math)[2]. The skinned primitive carries JOINTS_0 (a VEC4 of indices into skin.joints, not into the node array; mixing the two is a classic bug) and WEIGHTS_0. The skinned mesh node's own transform is ignored: the joint matrix is globalJoint · inverseBind, and the joints' global transforms already place the mesh in the scene[3]. (Older versions of the Khronos tutorial also multiplied by inverse(meshNodeGlobal) on the left, which cancels the mesh node's transform when the mesh is drawn with it.) LearnOpenGL's shader uses the same cheap mat3 cast for normals[7].
