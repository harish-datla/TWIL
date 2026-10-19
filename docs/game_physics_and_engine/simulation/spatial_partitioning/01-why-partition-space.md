# Why partition space

A level has 5,000 crates, barrels and ragdolls, and every frame the physics engine needs to know which ones are touching. Testing every object against every other is O(n²). A physics scene with 5,000 dynamic bodies produces 12.5 million pair tests per frame. At 60 fps on a single core that is roughly 750 million tests per second, each one requiring at least an AABB overlap check (up to six comparisons in 3D). Most of those pairs are nowhere near each other. The purpose of spatial partitioning is to skip them.

The split between broad phase and narrow phase exists for exactly this reason. The broad phase uses a spatial data structure to reduce n² candidate pairs to something closer to n or n log n, then the narrow phase runs the expensive per-pair geometry tests only on the surviving candidates.

The same structures serve other queries. Frustum culling walks the tree and prunes entire branches whose bounding volumes lie outside the camera frustum. Raycasting (mouse picking, line-of-sight checks, bullet traces) traverses the tree along the ray and skips volumes the ray does not intersect. Range queries ("give me every unit within 50 meters of this position") descend into nodes that overlap the query sphere and skip the rest. K-nearest-neighbor queries do the same with a shrinking search radius.

Without an acceleration structure, each of those queries is O(n). With one, the average case drops to O(log n) for tree-based structures or O(1) for hash-based structures, at the cost of O(n) or O(n log n) construction time and O(n) storage.

Worked example

Spread those 5,000 bodies evenly over a 1 km × 1 km map and drop a grid of 10 m cells on it: 10,000 cells, half a body per cell on average. Each body checks only its own cell and the eight around it, about 4.5 bodies, so the frame costs roughly 5,000 × 4.5 = 22,500 tests (each pair seen from both sides) instead of 12.5 million.
