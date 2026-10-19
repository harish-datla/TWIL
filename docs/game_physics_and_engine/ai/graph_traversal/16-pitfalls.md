# Pitfalls

Bugs that recur in graph-traversal code:

    Marking visited on expand instead of on discovery. The BFS bug: if you mark a vertex as visited when you pop it instead of when you push it, the same vertex can be pushed multiple times by different neighbors. The queue grows to O(|E|) entries instead of O(|V|), and on dense graphs that can exhaust memory.
    Recursive DFS on user content. Already covered in §5. The fix is the iterative form. If you keep the recursive form, set an explicit depth limit and fail loudly when it's exceeded.
    BFS on a weighted graph. BFS returns the path with the fewest edges, not the fewest weight units. If your edges have weights and you care about cost, use Dijkstra. In a game it shows up as an NPC taking the long, slow road because it has fewer waypoints.
    Dijkstra with negative weights. Dijkstra's correctness depends on the invariant that once a vertex is popped, its distance is final. A negative edge can update a finalized vertex, and Dijkstra never reconsiders it. Use Bellman–Ford if your weights are sometimes negative.
    A* with an inadmissible heuristic. If h(v) > h*(v) for some v, A* can return a suboptimal path. Often deliberate (weighted A* is faster), but should be explicit, not an oversight in the heuristic.
    Floating-point ties in shortest-path. Equal-cost paths are chosen by heap order, which shifts when the graph is edited or the neighbor order changes, and float rounding can differ across compilers and platforms. Break ties on a deterministic secondary key (lowest vertex id), or round costs to a stable precision.
    SCC report says "no cycle" but the build fails with a circular dependency. The graph the SCC pass runs on and the one the build system executes can disagree, for example when one is captured before a code-generation step and the other after. The SCC algorithm is correct; check that you're feeding it the post-codegen graph.
    Topological order isn't unique. Kahn's output depends on queue order, which depends on the initial scan order, which depends on how vertices were numbered. If you rely on a specific order (for reproducible builds, for the order in which render passes execute), replace the FIFO with a priority queue keyed on a stable id such as the asset path.
    Adjacency-list reallocation invalidating iterators mid-traversal. Calling graph.addEdge(...) while a BFS is in progress and the underlying std::vector<int> reallocates leaves the BFS iterating into freed memory. Either snapshot the graph, or queue the edits and apply them after the traversal finishes.

