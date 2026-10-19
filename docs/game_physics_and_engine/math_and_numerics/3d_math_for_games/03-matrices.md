# Matrices


A sword follows a hand, the hand follows an arm, and the arm follows a character walking through a level. Each link is a transform, and chaining them is matrix multiplication. A 4×4 matrix packs a 3×3 linear part (rotation, scale, shear) and a translation into one object that the GPU can apply to every vertex with a single multiply. Composing transforms is matrix multiplication, and the order matters: T·R ≠ R·T.

Worked example

R turns 90° about z, so it sends (1, 0, 0) to (0, 1, 0). T moves points 5 units along x. Apply both to the point p = (1, 0, 0), reading each product right to left.

T·R·p: rotate to (0, 1, 0), then move to (5, 1, 0). R·T·p: move to (6, 0, 0), then rotate to (0, 6, 0). Same two matrices, two different places.

The thing that actually causes bugs is that two independent conventions get conflated. Storage order (row- vs column-major) is how the floats sit in memory. Vector convention (column vs row vectors) is which side the matrix goes on. They are orthogonal: any of the four combinations is valid[2].
The number-one misconception

"Row-major means I pre-multiply" is false. Storage order does not dictate multiply order. glm and glam both store column-major and use column vectors (v′ = M·v)[3][4]. They are not contradicting each other; the two choices are simply independent.

Three facts follow, and each maps to a real bug:

    With column vectors, a model→world→view chain is written right-to-left, in the order the GPU applies it: Mview · Mworld · Mmodel · v. Read it as "model first."
    Translation lives in the 4th column under column vectors, the 4th row under row vectors. Put it in the wrong place and objects fly off to infinity.
    GPUs care about memory layout, not your math convention. GLSL matrices in a uniform block (a std140 block, GLSL's fixed-layout rule for uniform blocks, in Vulkan) default to column-major; upload row-major bytes without transposing, or without declaring the block row_major, and you silently transpose your transform.

One more scoping note: don't say "every engine stores column-major." The OpenGL and glTF lineage does, and so do glm and glam; DirectX's XMMATRIX is row-major with row vectors. Both are correct; they are different lineages[2].

Poke at it. The widget composes a transform from translation, rotation, and a scale that starts non-uniform (1.6 by 1), and prints the resulting column-major matrix. Flip the application order and the square shears; set both scales equal and the two orders agree again:
Rotate 30°
Scale X 1.60
Scale Y 1.00
T · R · S
T · S · R
The dashed outline is the original square, the filled shape the transformed one, and the two arrows its transformed x and y axes; the matrix beside them is the result. At the starting 30° and scale 1.6 by 1, T·R·S gives axis columns (1.39, 0.80) and (−0.50, 0.87), perpendicular. Switch to T·S·R and they become (1.39, 0.50) and (−0.80, 0.87), about 113° apart: a shear. With order T·R·S the square scales along its own axes, then rotates: a rectangle, no shear. With T·S·R the same non-uniform scale is applied after the rotation, so it stretches along world axes and the square shears into a parallelogram. Non-uniform scale and rotation don't commute, so the order changes the result; with equal X and Y scales it doesn't, and the toggle has no effect.
Compose a transform (column-major, column vectors)
C++
Rust

// glm: column-major storage, column vectors. Read right-to-left:
// scale first, then rotate, then translate.
glm::mat4 model = glm::mat4(1.0f);              // identity
model = glm::translate(model, position);          // T
model = model * glm::mat4_cast(rotation);        // R (quaternion → matrix)
model = glm::scale(model, scale);                 // S
// Equivalent: model = T * R * S; applied to a point as model * vec4(p, 1).

Check Your Understanding Q1 of 2

Your math library stores matrices column-major. Does that force you to pre-multiply (write v · M)?
No. Storage order and vector convention are independent; glm and glam are column-major and still use M · v.
Yes. Column-major storage requires row vectors and pre-multiplication.
Only on the GPU; on the CPU column-major always means M · v.
0/2 correct

Wrong answers, and why: column-major storage does not force row vectors (the conventions are independent); the transpose-on-upload symptom is a memory-layout mismatch, not a quaternion or near/far issue.

[Back to 3D Math for Games](index.md)
