# Loose bounds and margin

A crowd of NPCs shuffles a few centimeters every frame, and a tree that rebuilds a branch for every centimeter spends its frame on bookkeeping. Tight bounding volumes cause excessive re-insertion in dynamic scenes. When an object moves by one pixel, its AABB changes, and the tree must remove and re-insert it. If most objects move every frame, most leaves are removed and re-inserted every frame, which costs about as much as rebuilding the tree.

Loose octrees (Ulrich 2000[8]) expand each node's bounds by a factor of two. A node that would normally cover a 10x10 region instead covers 20x20, centered on the same position. The expansion guarantees that any object whose center is in the original region, and whose radius is at most half the original region size, fits entirely within the expanded bounds. Insertion needs no straddle test and no multi-cell registration: the object's radius picks the depth, and its center picks the node at that depth.

Fat AABBs in Box2D's dynamic tree[20] use a different approach. Each leaf's AABB is enlarged by a margin, and the object moves freely within the fat AABB without triggering a tree update. Only when the object's tight AABB leaves the fat AABB does the tree remove and re-insert the leaf with a new fat box. Box2D v2.4 used a 0.1 m margin and also extended the new box by four times the body's latest displacement, in the direction of motion, so a body moving steadily one way escapes it less often[24].

The trade-off: larger margins mean fewer re-insertions (good for update cost), but looser bounding volumes mean more false positives in queries (bad for query cost). Catto's GDC 2019 talk gives no formula for the margin: "There are many schemes for enlarging the AABB. Choose one that works well for your game."[20] Box2D v3 shows the balance moving with the rest of the design. Its margin is capped at 5 cm, and the source comment explains that benchmarks favored a small margin because it produces fewer contacts; the larger margin had existed to avoid insert and remove churn, which v3's broadphase no longer does, having moved to "a two-phase refit and rebuild setup similar to Jolt"[24].

Worked example

Under Box2D v2.4's rule, a body moving 5 cm per step gets a new fat box that reaches 0.1 m + 4 × 0.05 m = 0.3 m ahead of it. It crosses that in 0.3 ÷ 0.05 = 6 steps, so it's re-inserted every sixth step instead of every step. In a loose octree whose nodes are 10 m wide, each node's loose box is 20 m wide, so any object up to 5 m in radius whose center is in the node fits without straddling.
Check Your Understanding Q1 of 2

A dynamic AABB tree uses fat margins of 20 pixels. An object moves 2 pixels per frame. How many frames, on average, before the object triggers a re-insertion?
Roughly 1 frame, because the tree’s stored box no longer matches the object.
Roughly 10 frames: it takes 10 steps of 2 pixels to cross the 20-pixel margin.
Never, because a fat AABB is recentred on the object as it moves.
0/2 correct
