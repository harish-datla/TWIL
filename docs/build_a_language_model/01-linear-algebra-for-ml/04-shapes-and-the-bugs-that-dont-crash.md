# Shapes and the bugs that don't crash

Real model code juggles arrays with more than two axes. A batch of 32 sentences, each 128 tokens long, each token a 768-number vector, is one array of shape (32, 128, 768). Machine learning code calls an array like that a tensor, and its shape is the first thing to check when the numbers look wrong.

A very common operation is adding one vector to every row: a layer adds the same bias vector of 768 numbers to every token's output. Writing a loop for that is tedious, so array libraries let you write outputs + bias with shapes (32, 768) and (768,) and stretch the smaller array to fit. That stretching is broadcasting, and NumPy, PyTorch and JAX all use the same rule:[4]

    Line the two shapes up from the right.
    Compare sizes pair by pair. They must be equal, or one of them must be 1. A missing axis counts as 1.
    Any size-1 axis is stretched to match the other. Anything else is an error.

Worked example

(32, 768) + (768,): line up from the right. 768 matches 768. 32 is paired with a missing axis, which counts as 1 and gets stretched. Result: (32, 768), the bias added to every row, as intended.
Combine two shapes under the broadcasting rule
Shape of A
Shape of B
Reset
(3, 1) with (1, 4) → (3, 4): 12 output elements computed from 3 + 4 stored values. 3 vs 1 → 3; 1 vs 4 → 4.

Shapes line up from the right. A yellow outline marks a size-1 axis that gets stretched; a dot marks an axis one array doesn't have. The default pair, (3, 1) with (1, 4), builds a 3×4 result from only 7 stored numbers. Pick (3, 4) against (5, 4) to see an error. (3,) against (3, 4) is also an error, but only because the last sizes differ: against a 3×3 matrix the same vector would be stretched along the wrong axis with no complaint.

The dangerous cases are the ones that don't raise an error. Say you have 4 examples with 4 features each, shape (4, 4), and you want to scale each example by its own weight, a vector of shape (4,). Broadcasting lines the vector up with the features axis, so it silently scales each column instead of each row. Or add a (32, 1) column to a (1, 32) row and you get a 32×32 grid nobody asked for. The fix is cheap: reshape with explicit axes, such as (4, 1) for per-row weights, and assert shapes at function boundaries while developing.

Weight files have a shape trap too. PyTorch's linear layer stores its weight matrix as (outputs, inputs) and computes y = xWᵀ + b, the transpose of the Ax convention used above.[5] Other libraries store the other orientation. Load one into the other without transposing and the model runs, produces numbers, and is wrong.
Check Your Understanding Q1 of 2

A is 64×512 and B is 512×2048. How many multiply-adds does AB take, and what shape is the result?
64 × 512 × 2048 multiply-adds; shape 64 × 2048.
512 × 512 multiply-adds; shape 512 × 512.
64 × 2048 multiply-adds; shape 64 × 2048.
0/2 correct
