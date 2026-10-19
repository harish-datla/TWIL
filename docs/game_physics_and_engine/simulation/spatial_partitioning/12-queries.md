# Queries

The same tree serves the physics engine's overlap tests, the gun's raycasts, the renderer's culling and the AI's "who's nearby". All of these structures answer the same four query types, but each query traverses the tree differently:
Query	Test per node	Traversal	Engine use
Range (AABB overlap) 	AABB-AABB overlap (6 comparisons) 	Descend into every child whose AABB overlaps the query AABB. Prune the rest. 	Broad-phase collision, proximity triggers, area-of-effect abilities.
Ray 	Ray-AABB intersection (slab method) 	Stack-based DFS with ordered traversal (near child first). tmax tightens as closer hits are found. 	Mouse picking, line-of-sight, bullet traces, GPU ray tracing.
Frustum 	AABB-frustum test (6 plane tests, or SAT) 	Same as range query but with a frustum instead of an AABB. Entire subtrees outside the frustum are culled. 	Frustum culling for the render list. Runs every frame.
K-nearest-neighbor 	Distance to AABB (min-distance bounding) 	Priority queue ordered by minimum possible distance. Prune any node whose min-distance exceeds the current k-th nearest. 	AI perception ("find the 3 closest enemies"), flocking neighbors, point-cloud queries.

The key difference is how aggressively each query can prune. Ray queries get tighter with every hit (tmax shrinks). K-NN queries get tighter as the k-th neighbor gets closer. Range and frustum queries have fixed bounds and prune only by spatial overlap.

Worked example

An AI asks for its 3 nearest enemies. Midway through the search the third-nearest found so far is 12 m away. A node whose box is at least 15 m from the AI can't contain anything closer, so the whole subtree is skipped without opening it.
