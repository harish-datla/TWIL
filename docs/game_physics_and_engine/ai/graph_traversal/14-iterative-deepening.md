# Iterative deepening

A puzzle solver searching move sequences runs out of memory long before it runs out of time, because BFS keeps the whole frontier. BFS finds the shortest path but uses memory proportional to the frontier; for a uniform tree with branching b and goal at depth d, that's O(bd). DFS on a tree uses memory proportional to depth (O(d) for the recursive form) but doesn't find shortest paths.

Iterative-deepening DFS (Korf 1985[18]) gets both. Run depth-limited DFS with limit 1; if not found, throw it away and run again with limit 2; then 3; and so on. The repeated work looks wasteful but isn't: on a uniformly branching tree, the last iteration does most of the work (bd nodes), and all earlier iterations combined do less (the geometric sum is bd/(b−1)). For b = 10 the overhead is about 11%.

Worked example

With branching factor 10 and the goal at depth 5, BFS expands 1 + 10 + 100 + 1,000 + 10,000 + 100,000 = 111,111 nodes and holds a frontier of up to 100,000 in memory. Iterative deepening repeats the shallow levels on every pass and expands 1 + 11 + 111 + 1,111 + 11,111 + 111,111 = 123,456 nodes, only 11% more, while keeping just the 5 nodes of the current path in memory.

IDA* in the same Korf paper combines this with A*'s heuristic: at each iteration, the cutoff is on f(v) = g(v) + h(v), not depth. IDA* is a standard choice in classical-AI puzzle solving when the state space is too large for A*'s open set to fit in memory (sliding-tile puzzles, Rubik's cube, Sokoban). For game-engine navigation it's almost never the right choice: the navmesh fits in memory, and IDA* re-expands the same vertices on every iteration. Plain iterative deepening does ship in game AI, as the driver for alpha-beta search in board-game engines: searching depth 1, then 2, then 3 leaves a best move ready whenever the time budget runs out.
