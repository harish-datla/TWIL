# Building & Decomposing Transforms


An editor shows position, rotation and scale fields, the renderer wants one matrix, and a physics engine hands back a matrix you need to turn into fields again. Many engines store a local transform as translation, rotation (a quaternion), and scale, and build the matrix on demand. glTF defines a node's transform as exactly T·R·S with the rotation as a quaternion[7].

Composing is the easy direction, and it's what turns an editor's three fields into the one matrix the vertex shader applies:
eq. 7 · compose a transform M = T · R · S

In words: with column vectors you read the chain right to left, the order it actually runs: scale the object, then rotate it, then translate it into the world. Swap two of these and the result changes, because rotation and non-uniform scale do not commute (the §3 widget shows the shear).

Decomposing back out is where the edges are: translation is the last column, scale is the length of each of the three basis columns, and the rotation is what's left after dividing the columns by their scales.

Worked example

Build M = T(1, 2, 3) · R(90° about z) · S(2, 1, 1). Its first three columns are (0, 2, 0), (−1, 0, 0) and (0, 0, 1), and its last column holds the translation (1, 2, 3).

Decompose: the column lengths give scale (2, 1, 1); dividing each column by its length leaves (0, 1, 0), (−1, 0, 0) and (0, 0, 1), which is the 90° turn about z; and translation is (1, 2, 3). Everything comes back, because this matrix has no shear.
Trap: decomposition is lossy

Under shear or negative (mirrored) scale, you cannot cleanly split a matrix into rotation plus positive scale. glam's to_scale_rotation_translation documents only that the input "is expected to be a 3D affine transformation matrix otherwise the output will be invalid"[4]; a sheared matrix is still affine, so nothing flags it, and the rotation and scale that come back don't rebuild it. Two more from the same family: non-uniform scale doesn't commute with rotation, so T·R·S ≠ T·S·R; and lighting needs the inverse-transpose of the upper 3×3 (its inverse, flipped across the diagonal) to transform normals under non-uniform scale, not the model matrix itself.

Worked example

A 45° slope has tangent (1, −1, 0) and normal (0.707, 0.707, 0). Stretch it 2× along x with S = diag(2, 1, 1): the tangent becomes (2, −1, 0).

Transform the normal by S itself and you get (1.41, 0.71, 0), whose dot with the new tangent is 2.12, not 0: no longer perpendicular, so the lighting is wrong. The inverse-transpose is diag(0.5, 1, 1), giving (0.35, 0.71, 0), normalized (0.45, 0.89, 0), whose dot with (2, −1, 0) is 0.

[Back to 3D Math for Games](index.md)
