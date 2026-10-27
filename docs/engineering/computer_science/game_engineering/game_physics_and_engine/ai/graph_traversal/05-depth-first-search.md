# Depth-first search

Walking a scene graph to update transforms only needs to finish each branch before starting the next, and it should use as little memory as possible doing it. DFS swaps the queue for a stack and otherwise looks identical. The change in data structure changes the order of visits: instead of expanding outward in layers, DFS picks one neighbor, recurses into it, follows that branch as deep as it goes, and only backtracks when there's nothing left to expand. The order of discovery is a preorder traversal of a spanning tree rooted at the source.

DFS comes in two equivalent forms: recursive (uses the call stack as the data structure) and explicit-stack iterative (uses a std::vector as the stack). The recursive form is what algorithm textbooks show. The iterative form is what ships when the graph depth is user-controlled.

// Recursive DFS. Reads cleanly. Crashes on a deep enough graph.
//
// Each recursive call adds one stack frame (~64 bytes on x86-64 with no inlining).
// The default thread stack is 1 MiB on Windows, 8 MiB on Linux. Divide by 64,
// and your scene-graph traversal blows up around 16k to 130k deep.
void dfsRecursive(const CsrGraph& graph, int current,
                  std::vector<int>& visited)
{
    visited[current] = 1;
    for (int neighbor : neighbors(graph, current)) {
        if (visited[neighbor]) continue;
        dfsRecursive(graph, neighbor, visited);
    }
}

// Iterative DFS with an explicit stack. Same preorder as the recursive version,
// no recursion limit. The stack is a heap-allocated std::vector, so its size is
// limited by memory rather than by the thread stack; it can hold up to O(|E|)
// entries, because every expansion pushes all unvisited neighbors and stale
// duplicates wait until they are popped. This is the version that ships.
void dfsIterative(const CsrGraph& graph, int source,
                  std::vector<int>& visited)
{
    std::vector<int> stack;
    stack.push_back(source);

    while (!stack.empty()) {
        int current = stack.back(); stack.pop_back();
        if (visited[current]) continue;     // popped a stale entry
        visited[current] = 1;

        // Push neighbors in reverse so the leftmost neighbor is expanded first.
        // Without this, the order is reversed compared to the recursive version.
        auto adjacent = neighbors(graph, current);
        for (auto neighborIt = adjacent.rbegin(); neighborIt != adjacent.rend(); ++neighborIt) {
            if (!visited[*neighborIt]) stack.push_back(*neighborIt);
        }
    }
}

Both versions are O(|V| + |E|). The iterative version has one wrinkle: by the time you pop a vertex off the stack, it may have been visited by another path that pushed earlier. The if (visited[current]) continue; check guards that case. Forgetting it doesn't break correctness on a tree, but on a general graph with cross edges it visits the same vertex multiple times.
The three colors and edge classification

The three-color version of DFS, marking vertices white (unvisited), gray (on the recursion stack), or black (finished and popped), gives you something BFS doesn't: every edge gets a classification by when its other endpoint changed color[25]:

    Tree edge. Edge to a white vertex: the edge along which that vertex is discovered. The tree edges of a full DFS form the DFS spanning forest.
    Back edge. Edge to a gray vertex, which is always an ancestor in the DFS tree (or the vertex itself, for a self-loop). A directed graph has a cycle exactly when DFS finds a back edge. This is the textbook cycle detector, and a build system or render-graph compiler can use it to reject a cyclic dependency.
    Forward edge. Edge from a vertex to a black descendant in its own subtree. Only appears in directed graphs.
    Cross edge. Edge from one DFS subtree to a different, already-finished subtree. Only appears in directed graphs.

Maintaining the three colors is a two-line change: set the color to gray on entry to the recursive call and to black on exit. The cost is one extra write per vertex. The colors give cycle detection and topological sort (§8) directly; add discovery indices and a lowlink array and the same skeleton finds strongly connected components (§9), and, on undirected graphs, bridges and articulation points.
Recursion-depth crash

Recursive DFS over user-authored content can crash. A scene hierarchy nested as one long chain (a procedural placer that parents each new prop to the previous one builds exactly that) overflows the thread stack on the recursive traversal, while test scenes with shallow hierarchies never trigger it. Fix: switch to the iterative version, or set an explicit depth limit and return an error.
