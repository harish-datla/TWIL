# Surface area heuristic

The SAH cost model estimates the expected number of node intersection tests a random ray will perform. The key insight is that the probability a uniformly random ray hits a convex bounding volume is proportional to its surface area. The credit is usually given to Goldsmith and Salmon (1987)[5], who applied it to hierarchy construction; MacDonald and Booth's fuller derivation traces the surface-area-as-probability result itself back to Stone (1975)[6]. In 2D, surface area is the perimeter; in 3D, the actual surface area.

Worked example

A 2D node 10 × 10 holds ten objects: six in a tight 2 × 2 clump and four in a 3 × 3 clump. In 2D the "surface area" is the perimeter, so the node scores 40, the clumps 8 and 12. With traversal and intersection both costing 1, splitting around the clumps costs 1 + (8 ÷ 40) × 6 + (12 ÷ 40) × 4 = 3.4. Cutting the node into two 5 × 10 halves (perimeter 30 each) with five objects in each costs 1 + 0.75 × 5 + 0.75 × 5 = 8.5, and keeping it as a leaf costs 10. The heuristic picks the clump split, because a random ray is far less likely to hit small boxes.

For a node with bounding volume A containing children L and R, the formula below estimates what a split costs a random ray:
SAH cost C(L, R) = Ctrav + SA(L)SA(A) · NL · Cisect + SA(R)SA(A) · NR · Cisect

In words: pay once to visit the node, then pay for each child's objects weighted by the chance a ray that hits the parent also hits that child. SA(X) is the surface area of bounding volume X. NL and NR are the object counts in the left and right children. Ctrav is the cost of traversing one internal node; Cisect is the cost of intersecting one leaf object. The ratio SA(L)/SA(A) is the conditional probability that a ray hitting A also hits L.

The builder evaluates this cost for many candidate splits (every object boundary on each axis, or a fixed number of bins), picks the split with the lowest cost, and recurses. If no split is cheaper than making the node a leaf, the node becomes a leaf.

Binned SAH (Wald 2007[9]) approximates the exact evaluation by distributing objects into a fixed number of bins (typically 8 to 32) along each axis. The cost is evaluated at each bin boundary, not at every object boundary. This reduces the per-level cost from O(n log n) to O(n). In Wald's tests, a 252K-triangle animated model was rebuilt every frame in under 20 ms, which moved SAH quality into per-frame rebuilds.
Check Your Understanding Q1 of 2

The SAH cost function uses surface area as a proxy for ray-hit probability. What happens if you use volume instead of surface area?
Tree quality improves, because volume is a tighter bound on where the primitives are.
The cost function stops tracking geometric probability, so quality drops for non-cubic bounds.
Nothing changes, because surface area and volume scale identically under subdivision.
0/2 correct
