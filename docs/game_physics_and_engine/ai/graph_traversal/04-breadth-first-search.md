# Breadth-first search

A turn-based tactics game needs every tile a unit can reach in three moves, and the fewest moves to each. BFS visits vertices in increasing order of edge-distance from the source. Level 0 is the source. Level 1 is everything one edge away. Level 2 is everything two edges away that wasn't already on level 1. The data structure that enforces this order is a FIFO queue.

The three-color convention (white = unvisited, gray = in the queue, black = expanded) is the one Cormen, Leiserson, Rivest, and Stein's Introduction to Algorithms[25] uses for both BFS and DFS, and the one most textbooks now teach. The code below needs only two states (reached or not), because a gray vertex and a black one are handled the same way; the colors make the queue invariant easier to state.

// BFS. Returns parent[] and dist[]. Visits each vertex once, each edge once.
// Complexity: O(|V| + |E|), space O(|V|).
//
// 'parent[v] = -1' for unreached vertices; the source has parent = source.
// 'dist[v]' is the edge-count from source to v.
void bfs(const CsrGraph& graph, int source,
         std::vector<int>& parent,
         std::vector<int>& dist)
{
    const int vertexCount = vertexCountOf(graph);
    parent.assign(vertexCount, -1);
    dist  .assign(vertexCount, -1);

    std::queue<int> frontier;       // gray vertices: in the queue, not yet expanded
    parent[source] = source;        // sentinel: source is its own parent
    dist  [source] = 0;
    frontier.push(source);

    while (!frontier.empty()) {
        int current = frontier.front(); frontier.pop();
        // 'current' transitions from gray to black here.

        for (int neighbor : neighbors(graph, current)) {
            if (parent[neighbor] != -1) continue;   // already gray or black
            parent[neighbor] = current;
            dist  [neighbor] = dist[current] + 1;
            frontier.push(neighbor);                // newly gray
        }
    }
}

A few things are worth pointing out that the textbook usually skims past:

    The check is on discovery, not on expansion. Once a vertex is in the queue, it must not be added again, or the queue grows to O(|E|) entries instead of O(|V|). On dense graphs that can exhaust memory.
    BFS finds the shortest path in edge count, not in weight. Unweighted shortest path is BFS. Weighted shortest path needs Dijkstra (§10). BFS on a weighted graph silently ignores the weights and returns the path with the fewest edges.
    The complexity is O(|V| + |E|). Each reachable vertex is dequeued once and each of its edges is examined once (each undirected edge twice, once from each end), plus the O(|V|) initialization. On a CSR graph the edge reads are runs of contiguous memory, though the order in which runs are visited follows the queue.
    Reconstructing the path is a backward walk through parent from the goal until you hit the source. The path comes out reversed; either build a forward array and std::reverse it, or carry it on a stack as you walk.

The widget below paints a grid with random walls in BFS levels, moving only north, south, east, and west. Level 0 is the source; level k is every cell exactly k moves from it, so the wavefront grows in bands that flow around the walls. Each cell's color and number is its BFS dist[] value:
Live · BFS level expansion
▶ play
reset
step time 40 ms
level reached
0
cells expanded
0
queue peak
1
BFS spreading out from a source over a grid, one level at a time, with readouts for cells expanded, the level reached and the queue's peak size. Press play and slow the step time. On an open 4-connected grid, level k holds 4k cells: 4, 8, 12, 16, 41 cells within four steps. The queue holds the current wavefront, so its peak tracks the widest band. On an open 2D grid that band is the perimeter of the explored region, O(√n) for n cells explored: a couple of thousand cells on an open 1000×1000 grid. DFS's stack holds the current path, and on an open grid the dive snakes through most of the cells, so it can reach O(n). The case where BFS memory explodes is a tree search with branching factor b, where the frontier at depth d holds about bd nodes; that is the problem §14 (iterative deepening) solves.
Bidirectional BFS

If you know both endpoints (source s and goal g) you can launch two BFS frontiers, one from each, and stop when they meet[15]. The reason this is faster is geometric: on a uniform graph with branching factor b, single-direction BFS explores ~bd vertices to find a goal at depth d; bidirectional BFS explores ~2bd/2, about twice the square root of the single-direction count. The catch is implementation discipline: the two searches take turns (strictly alternating, or expanding whichever frontier is smaller), each newly discovered vertex is checked against the other direction's visited set, and the search finishes the current level before stopping, because the first meeting vertex found is not always on a shortest path (§12).
