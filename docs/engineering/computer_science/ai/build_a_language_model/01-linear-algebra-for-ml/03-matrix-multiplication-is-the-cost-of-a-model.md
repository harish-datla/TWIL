# Matrix multiplication is the cost of a model

Models process many inputs at once. Stack 32 input vectors as the rows of one matrix and a layer becomes matrix times matrix. For large language models, these matrix multiplications are where most of the arithmetic goes, so it pays to know their rules and their cost.

The rule: each entry of the result is the dot product of a row of the left matrix with a column of the right one. For that to work, a row of the left matrix must be as long as a column of the right one. The shared size gets summed away, and the outer sizes give the result's shape: (m×k) times (k×n) gives (m×n).[2]
C = AB,   Cij = Σp Aip Bpj

In words: to fill in row i, column j of the result, walk along row i of A and down column j of B, multiplying pairs and adding them up. Swapping the order, BA instead of AB, gives a different result and often a different shape.

The cost follows directly. The result has m·n entries and each one is a dot product of length k, so the whole thing takes m·n·k multiply-adds. Hardware specs usually count a multiply and an add as two FLOPs, giving 2mnk.

Worked example

A batch of 32 tokens, each a 768-number vector, goes through a layer that outputs 3072 numbers per token: (32×768) times (768×3072).

Result shape: 32×3072. Cost: 32 × 768 × 3072 ≈ 75 million multiply-adds, or about 151 million FLOPs, for one layer of one small model on one short batch.

The arithmetic is fixed by the shapes. What the programmer controls is the order the loops visit the numbers in, and that decides how fast the memory can keep up. Most languages and libraries, including C, C++, Rust, NumPy and PyTorch, store a matrix row by row by default.[3] A CPU fetches memory in chunks (64-byte cache lines on typical x86 chips), so reading the next entry along a row is nearly free, while jumping down a column means a fresh fetch every step.

The textbook loop order (row of A, column of B, then the inner sum) walks B down a column: bad. Swapping the two inner loops walks along rows of both B and the result: good. Same multiplications, same answer, different memory pattern. The memory hierarchy section of the containers page explains why it matters so much; the widget counts it.
Step through a product one multiply-add at a time
Loop order
Step
Play
Reset
A is 3×4, B is 4×5, C is 3×5: 60 multiply-adds in total, 0 done. Loop order ijk: press Step. Reads of B adjacent in memory to the previous read: 0 of 0.

A is 3×4 and B is 4×5, so C has 15 entries and takes 60 multiply-adds in either order. Yellow cells are the ones touched at the current step. In the textbook order the highlighted cell of B walks down a column, so no read of B is next to the one before it. In the row-friendly order it walks along a row, and 57 of the 59 consecutive reads of B are neighbors in memory. Both orders end with the same C. Play runs four steps per second.

Production libraries go much further: they cut the matrices into tiles small enough to stay in cache, reuse each tile many times, and use SIMD or GPU tensor-core instructions that do many multiply-adds at once. Loop order is the first step on that ladder, and on matrices bigger than the cache it alone commonly changes run time several-fold.
