# What's next

In this series the next page is Behavior Trees, the depth-first-walked tree that decides what an NPC does once it can find its way. Beyond this page, the graph algorithms worth knowing next:

    Best-first search beyond A*. D* (Stentz 1994) and D* Lite (Koenig and Likhachev 2002) repair a path incrementally when the graph changes instead of searching from scratch; Theta* (Daniel et al. 2010) finds any-angle paths on grids; SMA* (Russell 1992) searches within a fixed memory budget.
    Min-cost flow and max flow. Edmonds–Karp (BFS-based augmenting paths), Dinic, push-relabel. The graph-algorithm family that handles allocation problems: bandwidth, scheduling, network reliability.
    Minimum spanning trees. Kruskal (sort edges, union-find), Prim (grow one tree from a priority queue keyed on edge weight). Dijkstra's 1959 paper gives a Prim-style MST algorithm alongside the shortest path; both are in the same three-page note.
    Graph partitioning and clustering. METIS, KaHIP. For HPA*-style hierarchical pathfinding, scene-graph load balancing across worker threads, and asset-bundle planning.
    Parallel and GPU graph algorithms. Direction-optimizing BFS (Beamer et al. 2012; the thesis[24] covers it), which switches to having each unvisited vertex look for a parent in the current frontier when the frontier is large; GPU frameworks such as Gunrock and cuGraph. The same ideas apply when thousands of units share one flow field or distance map.
    Persistent data structures. Building a graph traversal that snapshots its visited set at each step for time-travel debugging. Debugging in release covers the broader topic.

For an engine programmer, a practical next step is to read two real implementations: your engine's render-graph compile step (culling, lifetime computation, and ordering, as in §15) and Detour's dtNavMeshQuery::findPath. Both are short, and both are built from the traversals on this page. 
