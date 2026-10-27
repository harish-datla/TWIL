# When to use which

Faced with a new graph problem, the first decision is which walk to use. The decision rule, ordered roughly by how often it comes up:
Question	Answer	Why
Shortest path in edge count, unweighted graph? 	BFS 	BFS visits in order of distance from source. The first time it reaches a vertex, that distance is final.
Shortest path with non-negative weights? 	Dijkstra (§10) 	BFS but the queue is a priority queue keyed by accumulated cost.
Shortest path with a geometry-based heuristic? 	A* (§11) 	Dijkstra with the priority bumped by an admissible estimate of remaining distance.
Detect a cycle in a directed graph? 	DFS 	A back edge (gray-to-gray) exists iff the graph has a cycle. One pass of DFS, one extra color array.
Order tasks so dependencies finish first (build system, render graph, animation blend)? 	Topological sort (§8) 	Kahn (BFS-flavored, indegree queue) or DFS postorder. Either works on a DAG.
Group nodes by mutual reachability (which assets form a cycle in your import chain)? 	Tarjan's SCC (§9) 	DFS with lowlink. Outputs the cycle membership for every node in one pass.
Memory-constrained tree search for an exact solution at large depth? 	Iterative deepening (§14) 	Repeated DFS at increasing depth limits. O(d) memory instead of BFS's O(b^d).
Just need to visit every reachable vertex? 	Either 	Both are O(|V| + |E|). Use DFS for cache locality on a flat array; use BFS for layer ordering.

Two patterns aren't on the table because they're more subtle. Tree problems where sibling order matters (behavior trees, UI hierarchies, the browser's DOM) want DFS preorder: parent, then children left to right. BFS gives layer order, which is almost never what UI or AI wants. Game-state search trees too wide to explore fully (chess, Go, turn-based strategy AI) want neither plain BFS nor plain DFS. They use alpha-beta pruning, a depth-first search that skips provably irrelevant branches, or Monte Carlo Tree Search, a best-first search guided by sampled playouts.
