# Collision detection: broad phase

With n bodies, testing every pair for collision is O(n2). At 5,000 bodies that is 12.5 million pairs per frame. The broad phase prunes the pair count to the small set of bodies whose bounding volumes actually overlap. A bounding volume here is usually an AABB, the smallest box with world-aligned edges that encloses the shape.

Worked example

Three crates span x = 0 to 2, 1 to 3, and 4 to 5. Sorted, the endpoints read: A starts (0), B starts (1), A ends (2), B ends (3), C starts (4), C ends (5). When B starts, A is still open, so (A, B) is a candidate pair; when C starts, nothing is open. One pair goes to the narrow phase instead of three, and with 5,000 bodies the all-pairs count would be 5,000 × 4,999 ÷ 2 = 12.5 million.

Three data structures dominate:

    Sweep-and-prune (SAP). Project every AABB onto one axis. Sort the endpoints. Walk the sorted list: when a "begin" endpoint is encountered, the body overlaps with every currently-open body. O(n log n) for the initial sort, close to O(n) per frame afterwards if bodies move slowly, because the list stays nearly sorted and insertion sort only pays for the endpoints that moved past each other. PhysX offers it as eSAP[12], and Bullet as btAxisSweep3.
    Spatial hashing. Hash each AABB into grid cells. Bodies in the same cell are candidate pairs. O(n) expected time. Good for uniform-size objects (particles, bullet casings). Poor when objects vary wildly in size (a character standing on terrain).
    Dynamic AABB tree. A bounding-volume hierarchy where each leaf is a body's AABB and internal nodes are the union AABB. Queries are tree traversals. Insertion and removal are O(log n) in a balanced tree. Box2D and Bullet's default broadphase use dynamic AABB trees, and Jolt uses a four-way variant (a quadtree of boxes)[9]. Good for scenes with mixed sizes and frequent insertion and removal.

sweep-and-prune broad phase
C++
Rust
Copy

// Sweep-and-prune on one axis.
// Returns candidate pairs whose AABBs overlap on the X axis.
// A second pass on Y (and Z for 3D) prunes further.
struct Endpoint {
    float value;       // x-coordinate of the AABB edge
    int   bodyIndex;   // which body this endpoint belongs to
    bool  isMin;       // true = left edge, false = right edge
};

std::vector<std::pair<int,int>> sweepAndPrune(
    std::vector<Endpoint>& endpoints)
{
    // insertion sort: O(n + swaps), so close to linear when the list is
    // still nearly sorted from last frame
    for (int i = 1; i < (int)endpoints.size(); ++i) {
        Endpoint key = endpoints[i];
        int j = i - 1;
        while (j >= 0 && endpoints[j].value > key.value) {
            endpoints[j + 1] = endpoints[j];
            --j;
        }
        endpoints[j + 1] = key;
    }

    // sweep: track which bodies are "open" (left edge seen, right edge not yet)
    std::vector<std::pair<int,int>> pairs;
    std::unordered_set<int> active;

    for (const auto& ep : endpoints) {
        if (ep.isMin) {
            // this body's AABB starts; it overlaps with every currently-active body
            for (int other : active) {
                int a = std::min(ep.bodyIndex, other);
                int b = std::max(ep.bodyIndex, other);
                pairs.push_back({a, b});
            }
            active.insert(ep.bodyIndex);
        } else {
            // this body's AABB ends
            active.erase(ep.bodyIndex);
        }
    }
    return pairs;
}

