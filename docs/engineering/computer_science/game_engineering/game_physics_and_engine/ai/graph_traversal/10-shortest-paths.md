# Shortest paths

BFS already solves the shortest-path problem when every edge costs the same: the first time it pops a vertex, that vertex is as close to the source as it will ever be. Edge weights break that guarantee. Give one edge a cost of 10 while its neighbors cost 1, and BFS, which counts edges and not cost, hands back a two-edge path costing 11 over a four-edge path costing 4. Dijkstra's algorithm is the fix, and A* (§11) is Dijkstra with a heuristic added to the priority.
What Dijkstra does

Dijkstra's algorithm[9] finds the lowest-cost path from one source to every other vertex, on a graph whose edge weights are all non-negative. It keeps one number per vertex, a tentative distance (0 for the source, ∞ for everything else), and a growing set of settled vertices whose distance is already final. Then it repeats a single move until no unsettled vertex is reachable:

    Settle the nearest. Take the unsettled vertex with the smallest tentative distance. Its tentative distance is now final: mark it settled.
    Relax its edges. For each neighbor reached by an edge of weight w, check whether (the settled vertex's distance) + w beats the neighbor's current tentative distance. If it does, lower the neighbor's distance and record the settled vertex as its parent.

That is the entire algorithm. Structurally it is BFS with the frontier changed from a FIFO queue to a min-priority queue keyed on tentative distance. BFS's FIFO queue always returns the vertex that has waited longest, which on a unit-weight graph is exactly the nearest unsettled vertex; keying on accumulated cost instead makes "nearest" mean cheapest-so-far rather than fewest-edges. Relaxing then does one thing BFS never has to: it can lower a vertex's tentative distance after first discovery, because on a weighted graph a cheaper route can surface later. Give every edge weight 1 and that never fires, so the priority queue hands vertices back in the same distance order BFS's FIFO queue does. That coincidence is what "Dijkstra is BFS with a priority queue" means.

Path reconstruction is the same backward walk BFS uses: each relaxation records a parent, and the shortest path to any vertex is its parent chain followed back to the source. With a binary heap for the priority queue the whole search is O((|V| + |E|) log |V|); a Fibonacci heap lowers the bound to O(|E| + |V| log |V|), and bucket or radix queues do better still when edge weights are small integers. In practice the Fibonacci-heap constants are bad enough that a plain binary heap usually wins anyway.
Why settling the nearest vertex is safe

Dijkstra rests on one claim: the moment you settle the unsettled vertex with the smallest tentative distance, that distance is already final. Nothing found later can beat it. Suppose a cheaper path to that vertex did exist. Trace it from the source; it has to leave the settled set somewhere, at some first still-unsettled vertex y. The cost to reach y along that path is at least y's own tentative distance, because the settled vertex just before y relaxed its edge to y when it was settled; y is unsettled, so its tentative distance is at least the one you are about to settle; and every edge past y adds a non-negative amount. The supposedly cheaper path therefore costs at least as much, which contradicts the assumption.[25] That single step is where the non-negative-weight requirement lives: one negative edge past y could drag the total back down, the guarantee fails, and you need Bellman–Ford instead (covered below, and flagged again in §16).

The widget runs Dijkstra on terrain with cost, moving only north, south, east, and west. Plain dark cells are cheap (entering one costs 1); the amber-hatched columns are slow (entering one costs 8), and each column has a short cheap gap. It settles one vertex per tick, tints each by its final cost from the source, and labels that cost when the cells are large enough to hold a number. The wavefront is a cost contour, not the even ring BFS painted in §4: it surges through cheap ground and lags against the slow bands, because a longer detour over cheap cells outranks a short hop through expensive ones.
Live · Dijkstra settling by accumulated cost
▶ play
reset
new terrain
step time 20 ms
vertices settled
0
frontier (heap) size
1
cost to goal
···
A grid with expensive cells, where each settled cell shows its cost from the source; the readouts give the settled count, the heap size and the cost to the goal. Press play, then regenerate the costs. Each settled cell's label is its final lowest cost from the source; once settled, it never changes, which is the invariant the safety argument above turns on. The contour is not round: a route of cheap cells that bends around a slow band beats the straight line through it, so the cheap corridors settle early and the slow bands settle late, at high cost. BFS (§4) on the same map would grow even diamond-shaped bands that ignore the terrain and hand back a path straight through the slow columns, because that path has fewer cells.

// Weighted CSR: the CsrGraph from §3 plus a per-edge weight array parallel
// to 'edges', so weights[i] is the cost of the edge stored at edges[i].
struct WeightedCsr {
    std::vector<int>    offsets;   // size |V| + 1
    std::vector<int>    edges;     // size |E|
    std::vector<double> weights;   // size |E|
};

// (neighbor, weight) pairs for vertex v. Allocates for exposition; hot-path
// code iterates the edge and weight spans directly instead.
inline std::vector<std::pair<int, double>> weightedNeighbors(const WeightedCsr& graph, int v) {
    std::vector<std::pair<int, double>> result;
    for (int e = graph.offsets[v]; e < graph.offsets[v + 1]; ++e)
        result.emplace_back(graph.edges[e], graph.weights[e]);
    return result;
}

// Dijkstra's algorithm with a binary heap.
// Returns a dist[] array; dist[v] = shortest cost from source to v, or +inf.
//
// Complexity: O((|V| + |E|) log |V|). With lazy deletion (below) a vertex is
// pushed once per successful relaxation, at most once per incoming edge, so the
// heap sees up to |E| + 1 insertions; log |E| <= 2 log |V|, so the bound holds.
// For dense graphs, an O(|V|^2) array scan in place of the heap is faster.
void dijkstra(const WeightedCsr& graph, int source,
              std::vector<double>& dist,
              std::vector<int>&    parent)
{
    const int vertexCount = static_cast<int>(graph.offsets.size()) - 1;
    dist  .assign(vertexCount, std::numeric_limits<double>::infinity());
    parent.assign(vertexCount, -1);

    using Entry = std::pair<double, int>;             // (cost, vertex)
    std::priority_queue<Entry, std::vector<Entry>, std::greater<Entry>> frontier;

    dist[source] = 0.0;
    frontier.emplace(0.0, source);

    while (!frontier.empty()) {
        auto [cost, current] = frontier.top(); frontier.pop();
        if (cost > dist[current]) continue;            // stale entry, skip

        for (auto [neighbor, weight] : weightedNeighbors(graph, current)) {
            double newCost = cost + weight;
            if (newCost < dist[neighbor]) {
                dist  [neighbor] = newCost;
                parent[neighbor] = current;
                frontier.emplace(newCost, neighbor);   // lazy: leave stale entries
            }
        }
    }
}

Three implementation details to get right or you'll have a slow or wrong Dijkstra:

    Lazy deletion. When a vertex's distance improves, the old entry stays in the heap. The if (cost > dist[current]) continue; skips it on pop. The alternative, finding and removing the old entry, needs a heap that supports decrease-key, which std::priority_queue doesn't.
    Negative weights kill correctness. Dijkstra's invariant ("the first time v is popped, dist[v] is final") relies on edges being non-negative. A negative edge can lower a "finished" vertex's distance after the fact. If your weights are sometimes negative, use Bellman–Ford.
    Floating-point ties. When two routes cost the same or nearly the same, heap order decides which one wins. On a grid with diagonal moves or a navmesh with floating-point edge lengths, a tiny change in the start position can flip the chosen route from one frame to the next, and the agent visibly dithers. Round costs to a stable precision, break exact ties on a deterministic secondary key (vertex id), or keep following the current path until a new one is cheaper by a margin.

The rest of the shortest-path family

Dijkstra is the default, but it is one member of a family. The others either restrict the weights to run faster (BFS, 0-1 BFS) or widen the problem to cases Dijkstra can't handle (negative weights, every pair of vertices):

    BFS is Dijkstra for unit weights, at O(|V| + |E|) with a plain FIFO queue instead of a heap. When every edge costs the same, the heap is wasted overhead.
    0-1 BFS handles edge weights that are only 0 or 1. Use a std::deque: push 0-weight neighbors to the front, 1-weight neighbors to the back. Still O(|V| + |E|), no heap needed.
    Bellman–Ford (Bellman 1958[8], Ford 1956[6], Moore 1959[7]) is the one to reach for when weights can be negative. Relax every edge |V|−1 times. O(|V| · |E|) worst case, and it detects negative cycles for free: if any edge still relaxes on the |V|th pass, a negative cycle is reachable from the source.
    Floyd–Warshall computes shortest paths between every pair of vertices in O(|V|³) time and O(|V|²) memory. For a small dense graph (a 16×16 cost table between AI tactical regions) it's the right tool; for a navmesh with 10,000 polys it's 1012 steps and a 100-million-entry table.

