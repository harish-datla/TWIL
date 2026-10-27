# Why traverse a graph

An NPC decides to walk to the bar, and before it takes a step the engine has to search a map of the level for a route. That search is a graph walk, and a modern engine's frame is a stack of them. A graph here just means things (vertices) joined by links (edges):

    The scene graph links each object to its parent; the engine walks it every frame to compute world transforms. With instancing it becomes a DAG, a directed graph with no cycles.
    The render graph[1] links rendering passes to the passes whose output they read; those links decide barrier placement, resource lifetimes and async-compute scheduling, and any valid pass order is a topological order, one that puts every dependency first (§8, §15).
    The asset import pipeline links source files and settings to the cooked files built from them; cooking visits it bottom-up so dependencies are baked before dependents.
    The navmesh links walkable polygons to their neighbors, and A* searches it whenever an NPC decides to walk.
    An AI behavior tree is walked depth-first from its root on each tick (event-driven implementations re-walk only the branches an event touched), and animation blend graphs are evaluated inputs-first. Even the shader cache walks each shader's include graph to compute the hash that keys it.

All of that runs on two procedures: breadth-first search (explore outward in rings from the start) and depth-first search (follow one path as far as it goes, then back up). Everything else is a specialization. Dijkstra is BFS with a priority queue. A* is Dijkstra with a heuristic. Topological sort is DFS with a postorder. Tarjan's SCC is DFS with a lowlink trick. Get the two base algorithms right and the rest reduces to "what data structure do I use for the frontier."
What this page covers

A working model of the graph walks an engine runs. Adjacency representations and when each one is right. BFS and DFS with the three-color convention, edge classification, and the recursion-depth bug that crashes the scene graph on deeply nested prefabs. Topological sort by Kahn's algorithm and by DFS postorder. Strongly connected components by Tarjan's lowlink. Dijkstra, Bellman–Ford, and the A* admissibility theorem. Bidirectional search, jump-point search, hierarchical pathfinding (HPA*). And the engine case studies: how Recast and Detour[2] build and search the navmesh that Unreal, Unity, and Godot use, why F.E.A.R. ran A* over actions as well as over its navmesh[3], and which graph walks a render-graph compiler runs.
The cost of the wrong traversal

The same walk can fit its budget or blow it, depending on how the graph sits in memory. A scene graph with 50,000 entities, traversed once per frame at 60 fps, gives you 333 ns per node if the traversal owns the entire 16.7 ms frame. It never does. Give the scene-graph pass a 1 ms budget (the dashed line in the widget below) and that leaves 20 ns per node to compute the world transform and push it onto the GPU upload buffer. A BFS using std::queue<Entity*> spends little on the queue itself; the cost is a DRAM miss on nearly every entity it dereferences, because the popped pointers arrive in an order that has nothing to do with where the entities sit in memory. One miss costs on the order of 100 ns, five times the per-node budget, before any real work. The same walk over a flat array of entities stored in preorder is a sequential scan: no allocation, no pointer chase, and the hardware prefetcher streams the next entities into cache ahead of the loop. The asymptotic class is the same; the constant factor decides whether the pass fits its budget.

The widget below puts the two on the same axis: entity count along X, microseconds along Y. The "pointer-chase BFS" curve assumes one DRAM-miss-equivalent per node (a hundred-ish nanoseconds on modern hardware, Norvig's ballpark[4]). The "flat-array DFS" curve starts at one L1 hit per node (about half a nanosecond, from the same table), which is the best case: small nodes, and a prefetcher that keeps up. Nodes that each fill a cache line or more make the scan bandwidth-bound and raise that figure; the third slider sets it. The low end of the pointer-chase slider stands for targets that hit in the L2 or L3 cache instead of DRAM:
Live · Traversal cost at frame scale
both
pointer chase
flat array
entity count 50,000
per-node cost, pointer chase 100 ns
per-node cost, flat array 0.5 ns
pointer-chase BFS
5000.0 µs
flat-array DFS
25.0 µs
1 ms scene-graph budget
1000 µs
Entity count runs across and microseconds up, with the 1 ms budget dashed; the readouts give each curve's cost at the chosen count. Drag the entity count and both per-node cost sliders. Both curves are linear in n, so the asymptotic class is identical; the slope is the cost of visiting one node. A BFS that pops entity pointers from a std::queue misses in DRAM on most of them because each popped entity is cold. A preorder-sorted flat array streams from cache. At the default settings the two differ by a factor of 200; at the pessimistic end of both sliders, by a factor of 2.
Check Your Understanding Q1 of 2

Your engine traverses a 50,000-entity scene graph once per frame and the pass costs 3.4 ms. Profiling shows 80% of that is cache misses on the node dereference. Which change reduces the time the most?
Switch the queue from std::deque-backed std::queue to a ring buffer.
Sort the entities into preorder traversal order in a flat array and walk it sequentially.
Parallelize the traversal across 8 cores.
0/2 correct
