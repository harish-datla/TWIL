# Dynamic updates

A physics scene has 3,000 bodies moving every frame, and the broadphase tree has to follow them. Static scenes build the acceleration structure once and query it many times. Dynamic scenes must update the structure every frame. Three strategies:

    Rebuild from scratch. Throw away the old tree, build a new one from the current object positions. Simple, and every frame gets a tree built for the current positions, but it costs O(n log n) per frame. It is practical while the build fits the frame budget, which fast builders such as binned SAH stretch to larger scenes.
    Refit. Walk the existing tree bottom-up and recompute each node's bounding volume from its children. O(n) per frame. Preserves the tree topology, so if objects move a lot the tree degrades (bounding volumes overlap, query performance drops). Refitting, with an occasional rebuild, is the usual treatment for deformable meshes in a BLAS.
    Incremental insert/remove. When an object moves, remove it from the tree, re-insert at the new position. Tree quality holds up if insertion chooses its spot well and repairs the tree on the way back up (rotations, as in Box2D's dynamic tree). Cost per update is about O(log n) for a reasonably balanced tree. The fat-AABB trick (§11) cuts the number of updates.

Erin Catto's GDC 2019 talk[20] lays out the remove-and-reinsert design Box2D used. Insertion picks the sibling whose pairing adds the least surface area to the tree, found with a branch-and-bound search after Bittner et al., instead of descending greedily. On the way back up, it refits the ancestor boxes and applies local tree rotations chosen to reduce surface area (after Kensler 2008), which also repairs the damage sorted input does to an incrementally built tree. The talk frames those rotations as the BVH counterpart of AVL rotations, but the goal is lower SAH cost, not equal subtree heights.

Worked example

For those 3,000 bodies, a rebuild costs on the order of 3,000 × log2 3,000 ≈ 35,000 steps per frame. Incremental updates with fat margins, if a tenth of the bodies leave their margin in a typical frame, cost about 300 × log2 3,000 ≈ 3,500. Refitting touches each of the tree's nodes once, a few thousand box updates, but never fixes a tree whose shape has gone stale.

The widget below shows 15 objects moving. Compare "fat AABBs" mode (objects move freely within their margins, and re-insertion happens only when the tight box leaves the fat box) with "tight AABBs" mode (the stored box is the object's own box, so every move forces a re-insertion). The re-insertion counters show the difference.
Live · Dynamic AABB tree: fat vs tight
fat AABBs
tight AABBs
total re-insertions
509
frames elapsed
883
re-inserts/frame
0.6/frame
reset
Fifteen moving objects with their stored boxes, and readouts for total re-insertions and re-inserts per frame. Switch between fat and tight AABBs and compare the counter. In fat-AABB mode, the dashed outlines are the fat boxes the tree stores: green while the object is still inside its margin, red on the frame the object escapes and is re-inserted. In tight-AABB mode the stored box is the object's own box, so every object is re-inserted every frame and the rate sits at 15 per frame. With the 12-unit margin, fat mode runs well under one re-insertion per frame.
Check Your Understanding Q1 of 2

Your physics engine runs at 60 fps with 3,000 dynamic bodies. The broadphase currently rebuilds a BVH from scratch every frame and takes 2.1 ms. Which change is most likely to reduce the cost?
Switch to a quadtree, which subdivides only where the bodies actually are.
Switch to an incremental dynamic AABB tree with fat margins, so most bodies skip re-insert.
Switch to a spatial hash, which rebuilds in O(n) instead of O(n log n).
0/2 correct
