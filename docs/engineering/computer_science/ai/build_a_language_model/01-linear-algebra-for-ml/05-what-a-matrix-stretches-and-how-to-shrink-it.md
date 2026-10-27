# What a matrix stretches, and how to shrink it

Two practical questions lead to the same idea. Why do signals in a deep network sometimes blow up or fade to nothing as they pass through layer after layer? And how can you fine-tune a model with billions of weights by training only a tiny fraction of them? Both answers come from asking which directions a matrix stretches, and by how much.

Start with a square matrix. Most input directions get both stretched and turned. A few special directions only get stretched: the output points the same way as the input. Those directions are the matrix's eigenvectors, and the stretch factor for each is its eigenvalue.[1]

Worked example

A = [[2, 1], [1, 2]].

A × (1, 1) = (2 + 1, 1 + 2) = (3, 3) = 3 × (1, 1). Same direction, 3 times longer: eigenvalue 3.

A × (1, −1) = (2 − 1, 1 − 2) = (1, −1). Unchanged: eigenvalue 1.

Now multiply some other vector by A again and again. Every pass triples the part of it that lies along (1, 1) and leaves the part along (1, −1) alone, so after a few passes the vector points almost exactly along (1, 1). That repeated multiplication is called power iteration, and it's a small model of a deep network. A signal passing through many similar layers gets pulled toward the direction those layers stretch most, and grows or shrinks by roughly that stretch factor at every layer. Stretch factors above 1 make signals explode; below 1 they fade. The training page shows how careful initialization keeps them near 1.
Multiply by a symmetric matrix until the direction settles
Starting angle (degrees)
170
Step
Play
Reset
After 0 multiplications the direction is (-0.9848, 0.1736); |cos| with the eigenvector (1, 1)/√2 is 0.573576. Rayleigh quotient vᵀAv / vᵀv = 1.657980; largest eigenvalue 3. Ratio of eigenvalues λ₂/λ₁ = 0.333 sets the convergence rate.

The matrix is the worked example, [[2, 1], [1, 2]]. Each step multiplies the current arrow by it and shrinks the result back to length 1. Brighter arrows are later steps; the dashed line is the (1, 1) direction. The right panel tracks the stretch estimate (the Rayleigh quotient), which climbs to 3. Start at 135° or 315°, exactly along (1, −1), and the arrow stays put for all 20 steps: rounding leaves a part of order 10⁻¹⁶ along (1, 1), and 20 triplings grow it by only 3²⁰ ≈ 3.5 × 10⁹.

Eigenvectors only exist for square matrices, and most weight matrices aren't square. The tool that works for every matrix is the singular value decomposition (SVD). It says that any matrix does three simple things in a row: turn the input, stretch it along the axes by a set of factors called the singular values, then turn the result.[1]

The useful reading for machine learning is the second form in the equation below. It splits the matrix into a sum of simple layers, each one a single column times a single row, weighted by its singular value and sorted biggest first. The big singular values carry most of the matrix; the small ones carry fine detail.
A = UΣVᵀ = Σi σi uiviᵀ

In words: the matrix equals turn, stretch, turn. Equivalently, it equals a stack of single-column-times-single-row pieces, each scaled by how much it matters. The number of pieces with a non-zero weight is the matrix's rank, the number of independent directions it really uses.

Keep only the k biggest pieces and you get a rank-k matrix that is the closest possible rank-k approximation of the original. That's the Eckart–Young theorem.[6] Storing it takes k(m + n + 1) numbers instead of m × n. For a 4096×4096 matrix at k = 16, that's about 131 thousand numbers instead of 16.8 million.
Keep k singular values of a 12×12 matrix
Rank kept, k
3
Noise
Reset
Rank 3 of 12: stores 75 numbers (k×(m+n+1)) instead of 144. Measured Frobenius error ‖A − A_k‖_F = 2.4181; √(σ²_{k+1}+…+σ²_12) = 2.4181. Singular values: 5.49, 2.53, 1.90, 1.45, 1.26, 0.81, 0.77, 0.67, 0.48, 0.41, 0.21, 0.00.

Left: a 12×12 matrix drawn as an image. Middle: the same matrix rebuilt from only its k biggest pieces. Right: the singular values, biggest first. The background, the bar and the corner block are simple shapes that the first few pieces capture. The diagonal band isn't: its singular values fall off slowly (5.49, 2.53, 1.90, 1.45, 1.26, …), so at k = 3 the band is still blurred and the error is 2.42. The error shown is the square root of the sum of the squared singular values thrown away. Large noise lifts most of the small singular values (σ₅ goes from 1.26 to 1.71), so the same error needs a larger k. The page computes the SVD itself with Jacobi rotations.
The precise statement, for the mathematically curious

This is the idea behind LoRA, the standard way to fine-tune a large model on modest hardware. Instead of updating a full 4096×4096 weight matrix, LoRA freezes it and trains a correction made of two thin matrices, 4096×16 and 16×4096. Their product has the full shape but only rank 16, and the two factors hold under 1% of the numbers.[7] The fine-tuning page measures what that small rank costs. The same decomposition also gives PCA, the classic technique for finding the few directions along which a dataset varies most.[1]
Check Your Understanding Q1 of 2

A LoRA update for a 4096×4096 weight matrix is the product of a 4096×16 and a 16×4096 matrix. How many numbers does it store?
16 × 16 = 256.
2 × 4096 × 16 = 131,072, about 0.8% of the full matrix.
4096 × 4096, because the product has that shape.
0/2 correct
