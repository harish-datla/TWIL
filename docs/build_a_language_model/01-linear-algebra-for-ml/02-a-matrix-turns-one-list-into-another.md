# A matrix turns one list into another

A layer in a neural network takes a vector in and produces a different vector out: in GPT-2 small, one layer turns each 768-number vector into a 3072-number one. The layer's weights are stored as a grid with one row per output and one column per input. That grid is a matrix, and running the layer means multiplying the matrix by the input vector.

Each output number is the dot product of one row with the input. So a layer is a stack of dot products, and each output asks the same question as a neuron: how much does this input line up with my row of weights?

Worked example

A = [[1, 2], [3, 4]] (two rows) and x = (5, 6).

Row 1 · x = 1×5 + 2×6 = 17. Row 2 · x = 3×5 + 4×6 = 39. So Ax = (17, 39).

Same answer read by columns: 5 × (1, 3) + 6 × (2, 4) = (5 + 12, 15 + 24) = (17, 39).

The column reading is the one that makes a matrix easy to picture. The first column, (1, 3), is where the input (1, 0) ends up; the second, (2, 4), is where (0, 1) ends up. Every other input is a mix of those two, so its output is the same mix of the columns.[1] That's why mathematicians call a matrix a linear map: it moves every point of space in a way that's fully decided by where a few basic directions land.
Ax = x1a1 + x2a2 + … + xnan

In words: the output is a blend of the matrix's columns, and the input's entries say how much of each column to use. Reading row by row gives exactly the same numbers.[2]

The widget below lets you edit a 2×2 matrix and see what it does to every point in the plane at once. Two numbers describe the result. The determinant says how much the matrix scales area; if it's zero, the matrix squashes the whole plane onto a line and the original input can't be recovered. The singular values say how much the matrix stretches along its strongest and weakest directions: a circle of inputs becomes an ellipse, and the ellipse's half-widths are the singular values.
Edit a 2×2 matrix and see what it does to the plane
Preset
a (row 1, col 1)
1
b (row 1, col 2)
0.5
c (row 2, col 1)
0.3
d (row 2, col 2)
1.2
Reset
Columns (images of e₁, e₂): (1, 0.3) and (0.5, 1.2). det = 1×1.2 − 0.5×0.3 = 1.050 (area factor 1.050). Singular values σ₁ = 1.5168, σ₂ = 0.6922; σ₁σ₂ = 1.0500. eigenvalues 1.500 and 0.700.

Left: the green and yellow arrows are the matrix's two columns, which is where (1, 0) and (0, 1) land. The purple parallelogram is where the unit square lands, and its area is |det|. Right: the unit circle lands on an ellipse whose half-widths are the singular values σ₁ and σ₂; their product equals |det|. The rank-1 preset flattens the plane onto a line, so σ₂ = 0 and the map can't be undone. Presets override the sliders; pick Custom to use them again.

Rotations are a special case worth remembering: they turn vectors without changing any length or any dot product, so a rotation matrix never loses or amplifies information.[1] Matrices with that property are called orthogonal. Transformers use exactly this property to tell the model where each word sits in the sentence: rotary position embeddings rotate each word's vectors by an angle that depends on its position, which changes their direction but never their size.
