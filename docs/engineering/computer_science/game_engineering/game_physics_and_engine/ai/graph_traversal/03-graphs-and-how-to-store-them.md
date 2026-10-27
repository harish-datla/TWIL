# Graphs and how to store them

A navmesh with ten thousand polygons, each touching three neighbors, can take 160 KB or 100 MB depending on how you store it. A graph is a set of vertices and a set of edges between them. Engine-side, vertices are entities, render passes, scene nodes, navmesh polygons; edges are parent-child links, resource dependencies, navmesh adjacencies, behavior-tree transitions. The shape of the graph and what you do with it dictates the storage:

    Directed vs undirected. An edge from u to v may or may not imply an edge from v to u. Scene graph: directed (parent → child). Navmesh adjacency: undirected (you can walk both ways). Render-pass dependency: directed and acyclic.
    Weighted vs unweighted. An edge may carry a cost: distance, time, fuel, an animation blend weight, a navmesh polygon area. Unweighted graphs use BFS to find the shortest path. Weighted graphs need Dijkstra (non-negative weights) or Bellman–Ford (any weights, including negative).
    Sparse vs dense. The classification is the ratio of |E| to |V|². A navmesh is sparse: a polygon has at most one neighbor across each of its edges (a handful per polygon), so |E| = O(|V|). A full complete graph like a "distance from every city to every city" matrix is dense: |E| = Θ(|V|²). The representation you pick has to match.

Three storage layouts cover every case you'll meet:
Layout	Storage	"Edge exists?"	"Neighbors of v"	When
Adjacency list 	Θ(|V| + |E|) 	O(deg v) 	O(deg v) 	Sparse graphs (most engine graphs). Default choice.
Adjacency matrix 	Θ(|V|²) 	O(1) 	Θ(|V|) 	Dense graphs, small fixed-size graphs (8×8 connectivity tables), Floyd–Warshall.
Edge list 	Θ(|E|) 	O(|E|) 	O(|E|) 	Bulk-edit, sorting edges by weight (Kruskal MST), input/output to disk.

Adjacency list is the default. Mehlhorn and Sanders[21] give the full asymptotic comparison; for sparse graphs the matrix wastes space at Θ(|V|²) when most cells are zero, and neighbor enumeration becomes Θ(|V|) per vertex (a full row scan) instead of Θ(deg v). Against a bit-packed matrix, a list with 32-bit neighbor indices breaks even on storage around |E| ≈ |V|² / 32; navmeshes and render graphs sit far below that density.

The cache-friendly adjacency list is not a std::vector<std::vector<int>>. It's two flat arrays in CSR layout: one offsets array and one edges array, both contiguous. Boost.Graph ships it as compressed_sparse_row_graph; SNAP[22], built for graphs that mutate, instead uses a hash table of nodes with sorted adjacency vectors, trading traversal speed for cheap edits:

// CSR adjacency list. Every traversal on this page runs on this layout.
struct CsrGraph {
    std::vector<int> offsets;   // size |V| + 1
    std::vector<int> edges;     // size |E|
};

// Neighbors of vertex v: contiguous span in 'edges'.
// data() + offset rather than &edges[offset]: a last vertex with no neighbors
// has offsets[v] == edges.size(), and indexing one past the end is UB.
inline std::span<const int> neighbors(const CsrGraph& graph, int v) {
    return { graph.edges.data() + graph.offsets[v],
             static_cast<size_t>(graph.offsets[v + 1] - graph.offsets[v]) };
}

// Vertex count, derived from the offsets array (one entry more than |V|).
inline int vertexCountOf(const CsrGraph& graph) {
    return static_cast<int>(graph.offsets.size()) - 1;
}

Traversal code favors CSR because every neighbor list sits in one contiguous array and the per-vertex index costs 4 bytes. A std::vector<std::vector<int>> spends a 24-byte vector header per vertex (on the common 64-bit standard libraries), so the index alone spans six times as many cache lines, and each neighbor list is a separate heap allocation placed wherever the allocator put it. Beamer's BFS measurements run on graphs stored in CSR[24], and in the June 2015 Graph 500[23] rankings he plots, shared-memory machines ran BFS at roughly 108 to 109 traversed edges per second per core.

The widget below shows the same 32-vertex graph stored three ways. Pick a vertex; the representation highlights the work each one does to answer "what are v's neighbors?"
Live · Three ways to store a graph
adjacency list
adjacency matrix
edge list
selected vertex 0
edge density 15%
storage
33 + 148 = 181 cells
cells read for lookup
7
neighbors found
5
One small graph shown three ways, with readouts for the selected vertex's neighbors, cells read and total storage. Switch between list, matrix and edges, then change the selected vertex and the edge density. The CSR adjacency list reads two offsets plus exactly deg(v) cells to enumerate v's neighbors. The matrix reads all |V| cells in v's row whether they are edges or not. The edge list reads every edge in the graph and tests each one. At 15% density most of the matrix row is empty cells; at 5% the row scan is nearly all waste, and the edge list is doing |E| comparisons for what should be O(deg v) work. Storage counts each undirected edge twice in the list (once per endpoint) and once in the edge list, which stores both endpoints.
