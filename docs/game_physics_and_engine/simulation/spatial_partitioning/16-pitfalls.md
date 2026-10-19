# Pitfalls

    Wrong structure for the workload. A BVH for a uniform particle system is slower than a grid. A grid for a scene with mixed-size objects (1-meter crates and 500-meter terrain chunks) hits the large-object problem. Match the structure to the data distribution and object-size variance.
    Deep trees from degenerate input. A quadtree with many objects at the same position keeps splitting, four children per level, and still ends with all of them in one leaf. Set a depth limit. BVHs have their own version: midpoint splits over exponentially spaced objects, or incremental insertion of sorted input with no rebalancing, can produce a tree as deep as a linked list. Median splits cannot (they halve the count every level), and SAH builds can stop and make a leaf when no split pays for itself.
    Large objects spanning many cells. In a grid, or a quadtree that registers objects in every leaf they overlap, an object that straddles four cells is stored in all four. An object that spans 100 cells is stored in 100. Loose bounds (§11) or hierarchical grids fix this.
    Update cost vs query cost. Fat margins reduce update cost but increase query cost (looser bounds, more false positives). Tight bounds reduce query cost but increase update cost (more re-insertions). The right margin depends on the ratio of queries to updates per frame and the expected object velocity.
    Forgetting to refit after deformation. If the underlying geometry changes (skeletal animation, cloth simulation, destruction) and the BVH is not refitted, queries return stale or incorrect results. Refitting is cheap (O(n)), but forgetting it is a common source of "physics not matching visuals" bugs.

Check Your Understanding Q1 of 2

Your open-world game has 200,000 static objects (buildings, rocks, trees) and 500 dynamic objects (NPCs, vehicles). Which setup is standard?
One BVH over all 200,500 objects, rebuilt from scratch at the start of every frame.
A static BVH built at load for the 200,000, plus a dynamic AABB tree for the 500; queries check both.
A single spatial hash holding all 200,500 objects, cleared and refilled each frame.
0/2 correct
