# Bounding volume hierarchies

A BVH is a tree of bounding volumes, and it's what a ray tracer uses to avoid testing a ray against a million triangles. Each leaf holds one object (or a small cluster). Each internal node holds the bounding volume that encloses all objects in its subtree. The bounding volume is almost always an AABB because AABB-AABB overlap is six comparisons and AABB-ray intersection (the slab method) is fast and branchless.

Construction is top-down or bottom-up. Top-down: start with all objects, pick a split (axis and position), partition into left and right, recurse. Bottom-up (agglomerative): start with one leaf per object, repeatedly merge the two nodes whose combined bounding volume is smallest. Top-down is simpler and faster to build. Bottom-up can produce tighter trees, but the naive version rescans every pair before each of its n−1 merges, which is O(n³); practical builders cache each cluster's nearest neighbor or search only nearby clusters to bring that down.

Worked example

A balanced BVH over 1,000 objects is about log2 1,000 ≈ 10 levels deep. A ray that misses the root's left child box discards 500 objects with one box test, and a ray whose path narrows cleanly to one leaf tests about two boxes per level, around 20, instead of 1,000 objects.

Top-down split strategies:

    Median split. Sort objects along the longest axis, split at the median. Produces balanced trees. Simple, O(n log n) per level.
    Midpoint split. Split at the spatial midpoint of the node's bounding volume. Faster than median (no sort), but can produce unbalanced trees if objects are clustered.
    SAH split. Evaluate a cost function at many candidate split positions, pick the one that minimizes expected traversal cost. The surface area heuristic (next section).

The widget below shows a BVH being constructed step by step. Each step picks an axis, evaluates a split, and partitions the objects. The bounding boxes of the resulting nodes are drawn.
Live · BVH construction (top-down, SAH)
▶ play
step
new scene
build step
0 / 13
nodes created
0
last action
ready
Rectangles are the objects and the colored boxes the BVH nodes, deeper levels warmer. Press step to watch one split at a time, or play, then new scene. Each step either creates a leaf (2 or fewer objects) or splits a group along the axis and position that minimizes the SAH cost. The outermost bounding box (the root) encloses all objects. Each subsequent split produces tighter child boxes. Randomize the scene to see how different object distributions produce different tree shapes.

In RTX hardware ray tracing, the BVH has two levels. The TLAS (top-level acceleration structure) stores instance transforms and pointers to BLASes. The BLAS (bottom-level acceleration structure) stores the actual triangle geometry for each mesh. Rebuilding the TLAS per frame is cheap (it contains only instance data). Rebuilding a BLAS means processing every triangle of the mesh, so static geometry builds the BLAS once and reuses it.
