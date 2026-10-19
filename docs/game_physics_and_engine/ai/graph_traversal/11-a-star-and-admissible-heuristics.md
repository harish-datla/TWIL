# A* and admissible heuristics

Dijkstra searches in every direction equally, so an NPC crossing a large map expands cells behind it as eagerly as cells toward its goal. A* is Dijkstra with one change: the priority isn't g(v) (cost from source) but g(v) + h(v), where h(v) is an estimate of the cost from v to the goal[13]. The estimate biases expansion toward the goal: vertices that look close get popped first. If the heuristic is good, A* expands a small fraction of the vertices Dijkstra would. If the heuristic is h(v) = 0, A* is exactly Dijkstra.

Worked example

On a 4-connected grid where every step costs at least 1, the Manhattan distance from (0, 0) to a goal at (10, 5) is 10 + 5 = 15. No path can take fewer than 15 steps, so the estimate never exceeds the real cost, which is what admissible means, and A* still returns an optimal path. Multiply it by 2, to 30, and A* reaches the goal faster but may return a longer path.

Heuristic best-first search already existed (the Graph Traverser program, which Nilsson first considered for Shakey, ordered its frontier by the estimate h alone); the 1968 paper added the cost-so-far term g to the priority and proved when the result is optimal:
Hart, Nilsson, Raphael 1968, the admissibility theorem (informal)

If the heuristic h is admissible (never overestimates the true remaining cost h*(v)), then A* with that heuristic finds an optimal solution.

If the heuristic is also consistent (sometimes called monotone): h(v) ≤ cost(v, u) + h(u) for every edge v→u, then the first time A* expands a vertex it has already found the cheapest path to it, so no vertex needs expanding twice. Under that assumption the paper also proves A* optimally efficient: with suitable tie-breaking, no algorithm using the same heuristic expands fewer vertices. The 1972 SIGART correction[14] claimed consistency wasn't needed for that result; Dechter and Pearl (1985) showed it is, with an admissible but inconsistent heuristic on which A* expands arbitrarily more vertices than a competing algorithm[32].

The consistency condition decides what an implementation has to do with closed vertices (ones already expanded). With a consistent heuristic, a closed vertex can be skipped forever, which is what the widget below does. With a heuristic that is admissible but not consistent, A* can later find a cheaper path to a closed vertex; to keep the optimality guarantee it must reopen that vertex and expand it again. Detour's findPath reopens closed polygons whenever a cheaper route turns up[31].

On a 2D grid, the canonical admissible heuristics are:

    Manhattan distance |Δx| + |Δy|: admissible for 4-connected grids (only N/S/E/W moves).
    Octile distance max(|Δx|, |Δy|) + (√2−1) · min(|Δx|, |Δy|): admissible for 8-connected grids with diagonal cost √2.
    Euclidean distance √(Δx² + Δy²): admissible for any grid; loose for the 4- and 8-connected cases. Use only when the grid allows arbitrary-angle moves (rare in tiles, common on navmesh).

On a navmesh, the usual heuristic is straight-line distance from the search node to the goal. No path between two points is shorter than the straight line, so it is admissible as long as no area costs less than 1 per unit of distance. Detour places each search node at the midpoint of the polygon edge it was entered through, charges distance times the area's cost, and scales the straight-line heuristic by 0.999, just under 1[31]. Multiply the heuristic by a constant w > 1 (weighted A*) and A* usually expands fewer nodes, but it gives up the optimality guarantee: with an admissible h, the path it returns costs at most w times the optimum. It is a common speed/quality knob in game pathfinding[26].

The widget runs Dijkstra and A* side by side on the same 8-connected grid (straight moves cost 1, diagonals √2) with the same source and goal, using the octile heuristic. The heuristic-weight slider goes from 0 (= Dijkstra) past 1 (admissible) to 2.5 (greedy and possibly suboptimal). "Run" replays both searches one expansion per tick; moving the slider shows the finished result immediately:
Live · Dijkstra vs A* with a heuristic-weight slider
run
new obstacles
heuristic weight 1.0
Dijkstra expanded
498
A* expanded
50
Dijkstra path
27.83
A* path
27.83
Dijkstra and A* on the same grid, with each one's expanded count and path cost. Run them, then drag the heuristic weight from 0 past 1 and run again. At weight = 0 the two grids match: zero heuristic means A* is Dijkstra. At weight = 1 A* expands a slimmer corridor toward the goal but the path length matches Dijkstra's exactly: optimal, with fewer expansions. Past weight = 1 the heuristic over-estimates and the path A* returns can be longer than Dijkstra's (never more than w times as long): A* commits to the goal direction before exploring detours that would have been cheaper. On the default map the path grows from 27.83 to 28.66 at weight 1.1 and to 31.14 at 2.5, while the expanded count drops from 50 to 28.
