# The navigation mesh

The guard needs to know where it can stand and how those places connect, in a form a search can walk quickly. A navmesh represents the walkable surface as convex polygons that tile it. Adjacent polygons sharing an edge are linked into the dual graph: each polygon is a node, each shared edge a link[1]. Convexity is load-bearing: for any two points inside one convex polygon, the straight segment between them stays inside, so movement within a polygon is collision-free by construction and needs no search. Search only happens between polygons.

Worked example

An L-shaped corridor can't be one convex polygon: a straight line from the end of one arm to the end of the other cuts across the inside corner, through the wall. Split it at the corner into two rectangles that share an edge and both pieces are convex. The dual graph has 2 nodes and 1 link, and an agent anywhere in either rectangle can walk straight to any other point in the same rectangle.
It's convex polygons (not just triangles), and it's per-agent

Triangles are one tessellation; Recast (§3) emits convex polygons of up to 6 vertices (Detour's per-polygon limit), not only triangles. The navmesh is an abstraction of the walkable surface for a specific agent radius and height: a fatter agent needs a different, more inset navmesh, so it isn't "the floor geometry." The polygon-adjacency links are undirected; only off-mesh links (jumps, ladders, drops) can be one-directional.
The navmesh polygon and its dual graph
C++
Rust

// Each polygon stores its vertices AND, per edge, the neighbor polygon across
// that edge (or NO_NEIGHBOR for a wall). The neighbor array IS the dual graph A* walks.
constexpr int MAX_VERTS   = 6;    // Detour's DT_VERTS_PER_POLYGON
constexpr int NO_NEIGHBOR = -1;

struct NavPoly {
    std::array<int, MAX_VERTS> vertIndex;   // indices into the shared vertex pool
    std::array<int, MAX_VERTS> neighbor;    // neighbor poly per edge, or NO_NEIGHBOR
    int vertCount;                          // 3..6 (Recast emits up to 6-gons)
};
struct NavMesh {
    std::vector<Vec3>    vertices;             // shared vertex pool
    std::vector<NavPoly> polys;                // nodes of the dual graph; neighbor[] feeds A*
};

