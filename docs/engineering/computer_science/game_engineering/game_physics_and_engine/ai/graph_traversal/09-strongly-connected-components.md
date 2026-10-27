# Strongly connected components

The asset cooker reports "circular dependency" and stops, and the artist needs to know which files form the loop. A strongly connected component is a maximal set of vertices where every pair can reach each other via directed edges. In a DAG, every SCC is a single vertex (no cycles). In a general directed graph, every cycle lies inside one SCC, and cycles that share a vertex merge into the same SCC; contracting each SCC to a single vertex leaves a DAG. Engine relevance: when a build system, asset cooker, or render graph reports "circular dependency," the useful message is "these SCCs have more than one node, and here are their members." A back edge (§5) proves that a cycle exists; the SCC algorithm lists every vertex caught in one.

Two algorithms ship. Tarjan's (1972[16]) runs in one DFS pass, maintaining a discovery index per vertex and a "lowlink": the smallest discovery index the vertex's DFS subtree reaches through one edge to a vertex still on the component stack. A vertex whose lowlink equals its own discovery index is the root of an SCC, and it plus everything above it on the stack form the component. Kosaraju–Sharir's (Kosaraju 1978 unpublished; Sharir 1981[17]) runs two DFS passes: one to get a finish-order on the original graph, a second on the transposed graph in reverse finish-order, where each DFS tree is one SCC. Both are O(|V| + |E|). Tarjan's is faster in practice because it touches the graph once and skips the transpose; Kosaraju's is shorter to write.

// Tarjan's SCC algorithm. One DFS pass, output one component per call to onComponent.
// 'discovery' is the order each vertex was first seen; 'lowlink' is the smallest
// discovery index this vertex's subtree reaches through one edge to a vertex
// that is still on componentStack.
void tarjanScc(const CsrGraph& graph, std::function<void(std::span<const int>)> onComponent) {
    const int vertexCount = vertexCountOf(graph);
    std::vector<int>  discovery(vertexCount, -1);
    std::vector<int>  lowlink  (vertexCount, -1);
    std::vector<bool> onStack   (vertexCount, false);
    std::vector<int>  componentStack;
    int nextIndex = 0;

    auto strongconnect = [&](auto& self, int current) -> void {
        discovery[current] = nextIndex;
        lowlink  [current] = nextIndex;
        nextIndex++;
        componentStack.push_back(current);
        onStack[current] = true;

        for (int neighbor : neighbors(graph, current)) {
            if (discovery[neighbor] == -1) {
                self(self, neighbor);                       // tree edge: recurse
                lowlink[current] = std::min(lowlink[current], lowlink[neighbor]);
            } else if (onStack[neighbor]) {
                // Edge back into an SCC still under construction (a back edge,
                // or a cross edge to a vertex whose component isn't finished).
                lowlink[current] = std::min(lowlink[current], discovery[neighbor]);
            }
            // Else: the neighbor's SCC is already complete and emitted. Skip.
        }

        // If 'current' is the root of an SCC, pop it off the stack.
        if (lowlink[current] == discovery[current]) {
            std::vector<int> component;
            while (true) {
                int popped = componentStack.back();
                componentStack.pop_back();
                onStack[popped] = false;
                component.push_back(popped);
                if (popped == current) break;
            }
            onComponent(component);
        }
    };

    for (int v = 0; v < vertexCount; ++v)
        if (discovery[v] == -1) strongconnect(strongconnect, v);
}

What's intentionally missing

The recursive form blows up on deep graphs the same way recursive DFS does. Production code uses an explicit stack with a small state machine; there are well-tested iterative versions in the LLVM (used to identify SCCs in the call graph for inlining) and Boost.Graph codebases. The version above is the textbook one. Use the iterative form in shipping code, especially when the graph can be authored by users.
