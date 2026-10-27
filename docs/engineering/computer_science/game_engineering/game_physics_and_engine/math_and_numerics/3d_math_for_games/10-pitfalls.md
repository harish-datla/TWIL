# Pitfalls


Most bugs in this tutorial are convention mismatches. Symptom first, then the convention behind it:
Whole scene upside down
Vulkan's Y-down NDC, no Y-flip in the projection or viewport.
Model mirrored, faces inside out
Handedness mismatch on import flips an axis and inverts winding.
Objects fly to infinity
Translation written into the wrong row/column for your vector convention.
Z-fighting in the distance
Standard depth mapping crowds precision near the camera; reversed-Z with a floating-point depth buffer spreads it out.
Geometry clipped close to the camera
An OpenGL −1…1 projection fed to Vulkan's 0…1 clip volume.
Object shears when scaled
Non-uniform scale composed in the wrong order, or decomposed from a sheared matrix.
Camera spins the long way
SLERP without the q₀·q₁ < 0 shortest-path check.
NaNs spreading through transforms
Normalizing a zero-length vector.
Lighting wrong on scaled meshes
Transforming normals with the model matrix instead of its inverse-transpose.

[Back to 3D Math for Games](index.md)
