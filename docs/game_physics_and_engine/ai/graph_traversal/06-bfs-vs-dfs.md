# BFS vs DFS, side by side

The shape of the frontier is the visible difference. BFS expands layer by layer; DFS spelunks one branch. Both visit every reachable vertex exactly once. Both are O(|V| + |E|). They give different answers to "what path gets from s to g": BFS returns the path with the fewest edges; DFS returns whichever path its dive happened to follow, often a long one. The widget runs both on the same labyrinth, same source, same goal, one expansion per tick each:
Live · BFS vs DFS on the same maze
▶ play
reset
new maze
step time 35 ms
BFS expanded
0
BFS path length
···
DFS expanded
0
DFS path length
···
The same maze twice, BFS on one side and DFS on the other, with each side's expanded count and path length. Press play, then regenerate the maze a few times and compare the path lengths. BFS stops when it pops the goal off its queue; its path is the shortest in edge count. DFS stops when it pops the goal off its stack; its path is the branch its dive followed. The goal sits on the far side, so BFS has to expand nearly every cell closer to the source than the goal before reaching it, while DFS either dives toward the goal early or wastes its time in dead ends first. Over 300 random mazes of this size, DFS expanded fewer cells than BFS in about three out of four, but its path averaged about 45% longer. Eight interior walls are knocked out after the maze is carved; without them the maze is a tree with exactly one path to the goal, and both would return it.
Check Your Understanding Q1 of 2

You're writing the runtime that picks the navmesh polygon a unit should walk through next. The graph has up to 50,000 polys and edge weights matter (some polys are slow water, some are fast road). Which algorithm do you implement?
BFS, because it always finds the shortest path.
DFS, because it uses less memory.
A* with straight-line distance, scaled by the cheapest per-meter terrain cost, as the heuristic.
0/2 correct
