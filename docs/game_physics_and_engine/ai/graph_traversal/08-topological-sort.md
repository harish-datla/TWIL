# Topological sort

A DAG always has a linear order on its vertices that respects every edge: every dependency before every dependent. The classic example is a build system: compile util.cpp before main.cpp if main depends on it. The classic engine example is the render graph: the depth pre-pass must finish before the opaque pass that depth-tests against its output. Topological sort produces that order.

Worked example

Four passes: Depth feeds Opaque and SSAO, and both feed Composite. Counting incoming edges gives Depth 0, Opaque 1, SSAO 1, Composite 2. Kahn's algorithm emits Depth (the only one at 0), which drops Opaque and SSAO to 0; it emits those two, which drops Composite to 0; then it emits Composite. The order Depth, Opaque, SSAO, Composite runs every pass after everything it reads.

Two algorithms exist. Kahn's algorithm (1962[12]) is the BFS-flavored one: maintain an indegree count for each vertex, push every zero-indegree vertex onto a queue, pop and emit, decrement the indegree of each neighbor, push any neighbor whose indegree just hit zero. The DFS-postorder variant (CLRS chapter 20[25]; an early description in print is Tarjan's 1976 paper[34]) runs DFS and emits each vertex when it turns black; the reversed sequence is a topological order.

// Kahn's topological sort. Returns an order respecting every edge,
// or an empty vector if the graph has a cycle.
//
// Complexity: O(|V| + |E|). Each vertex is enqueued once, each edge is
// inspected once when its source is processed.
std::vector<int> topoSortKahn(const CsrGraph& graph) {
    const int vertexCount = vertexCountOf(graph);
    std::vector<int> indegree(vertexCount, 0);

    // First pass: compute every vertex's indegree.
    for (int v = 0; v < vertexCount; ++v)
        for (int neighbor : neighbors(graph, v))
            indegree[neighbor]++;

    std::queue<int> ready;       // vertices with no remaining prerequisites
    for (int v = 0; v < vertexCount; ++v)
        if (indegree[v] == 0) ready.push(v);

    std::vector<int> order;
    order.reserve(vertexCount);

    while (!ready.empty()) {
        int current = ready.front(); ready.pop();
        order.push_back(current);

        for (int neighbor : neighbors(graph, current)) {
            // 'Consume' the edge from current to neighbor: one fewer prerequisite.
            if (--indegree[neighbor] == 0) ready.push(neighbor);
        }
    }

    // Any vertex never emitted is on a cycle or downstream of one.
    if (static_cast<int>(order.size()) != vertexCount) return {};
    return order;
}

Kahn's algorithm detects cycles for free: if the loop exits with vertices still unprocessed, each of them is on a cycle or downstream of one. That leftover set is a superset of the cycle, so an error message built from it names innocent dependents too. To name only the cycle, run an SCC pass (§9) over the leftovers, or use the DFS version: when it finds a back edge u→v, the gray vertices on the recursion stack from v down to u are exactly one cycle, ready to print as "circular dependency: A → B → C → A".

The widget runs Kahn's on a small render graph. Each node shows its indegree counter (how many of its prerequisites haven't been emitted yet); only zero-indegree nodes can be emitted. The order it produces is one valid topological order; there are usually many, and Kahn's picks the one its queue order dictates, which depends on the order of the initial scan:
Live · Kahn's topological sort
step ▸
▶ play
reset
add cycle
step time 600 ms
emitted
0
queue size
2
status
ready
A render graph's passes with their incoming-edge counts, the queue of passes ready to run, and the emitted order. Step or play the sort, then press add cycle and run it again. "Add cycle" inserts an edge from Composite back to GBuffer. Everything on that loop, and everything downstream of it, never reaches indegree zero, so the sort stops after emitting ShadowMap, with eight vertices left. Six of them (GBuffer, DepthPyramid, Lighting, SSAO, SSR, Composite) lie on loops through the new edge and form one strongly connected component; Bloom and ToneMap are only downstream of it. Telling those two groups apart is the job of the SCC pass in §9.
