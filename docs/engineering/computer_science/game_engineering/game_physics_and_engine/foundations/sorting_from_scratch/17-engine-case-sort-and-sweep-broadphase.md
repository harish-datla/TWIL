# Engine Case: Sort-and-Sweep Broadphase


Collision detection's broadphase finds candidate pairs of objects whose AABBs overlap. The naive algorithm is n²; sweep-and-prune reduces it to roughly n log n per frame by sorting AABB endpoints along an axis and sweeping a moving "active list" across them[29].

The sort here is interesting because the input is almost sorted from frame to frame. Object positions change a little per frame; the endpoint list mostly preserves order from the previous frame. Insertion sort costs O(n + s) for s out-of-order pairs, which beats a general n log n sort on this workload. With 10,000 endpoints of which 200 pairs swapped order since last frame, that's about 10,000 + 200 = 10,200 steps, against roughly 133,000 comparisons (10,000 × log2 10,000) for a sort from scratch. Incremental sweep-and-prune implementations exploit it: Bullet's btAxisSweep3 keeps last frame's sorted endpoint arrays and moves each changed endpoint up or down into place, updating the overlap pairs as endpoints pass each other[33].

Some physics engines skip sort-and-sweep entirely. Box2D's broadphase uses b2DynamicTree, an incrementally updated AABB tree; version 3 picks each new leaf's sibling and its tree rotations by surface area (perimeter, in 2D)[34], the cost model BVH builders use[30]. The ordering work doesn't disappear; it moves inside the data structure, and the tree's incremental updates fill the role of an explicit per-frame sort.

[Back to Sorting Algorithms](index.md)
