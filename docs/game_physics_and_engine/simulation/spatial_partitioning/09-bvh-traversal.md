# BVH traversal

A player fires a hitscan rifle, and the game needs the nearest thing along that line out of tens of thousands of objects. The canonical BVH traversal for ray queries is a stack-based depth-first search. It pushes the root, then loops: pop a node and test the ray against its AABB; on a miss, discard it; on a hit at a leaf, test the ray against the leaf's objects; on a hit at an internal node, push both children. The stack never holds more entries than the tree is deep plus one, and a million-primitive tree is at least 20 levels deep (log2 of a million), more when SAH builds it unbalanced.

The slab method for ray-AABB intersection computes entry and exit times on each axis independently, then checks if the intervals overlap. It is branchless, maps well to SIMD (instructions that do the same operation on several values at once), and is the standard in production ray tracers.

Worked example

A ray starts at (0, 0) heading (1, 0.5). Against a box spanning x = 2 to 4 and y = 0 to 3, the x slab gives t = 2 to 4 and the y slab t = 0 to 6. The latest entry is 2 and the earliest exit is 4, and since 2 < 4 the ray hits, entering at t = 2. Raise the box to y = 2.5 to 3 and the y slab becomes t = 5 to 6: entry 5 comes after exit 4, so it's a miss.
Ray-AABB intersection (slab method)
C++
Rust

#include <algorithm>

struct Vec3 { float x, y, z; };
struct AABB { float minX, minY, minZ, maxX, maxY, maxZ; };

// Slab-method ray-AABB intersection.
// Returns true if the ray [origin, origin + dir * tmax] hits the box.
// On hit, tmin is the entry distance (may be negative if origin is inside).
bool rayAABB(const Vec3& origin, const Vec3& invDir,
             const AABB& box, float& tmin, float tmax)
{
    // invDir = 1.0 / direction, precomputed to avoid division per test
    float t1 = (box.minX - origin.x) * invDir.x;
    float t2 = (box.maxX - origin.x) * invDir.x;
    tmin = std::min(t1, t2);
    tmax = std::min(tmax, std::max(t1, t2));

    t1 = (box.minY - origin.y) * invDir.y;
    t2 = (box.maxY - origin.y) * invDir.y;
    tmin = std::max(tmin, std::min(t1, t2));
    tmax = std::min(tmax, std::max(t1, t2));

    t1 = (box.minZ - origin.z) * invDir.z;
    t2 = (box.maxZ - origin.z) * invDir.z;
    tmin = std::max(tmin, std::min(t1, t2));
    tmax = std::min(tmax, std::max(t1, t2));

    return tmax >= std::max(tmin, 0.0f);
}

Two optimizations matter in practice:

    Ordered traversal (near child first). When both children are hit, push the far child first and the near child second. The near child is popped and tested first, and any hit in the near child tightens tmax, so when the far child is popped its entry distance often lies beyond the closest hit and the whole subtree is skipped. The saving applies to closest-hit queries; an any-hit shadow ray terminates at its first hit instead of tightening tmax.
    Packet traversal. Test a bundle of coherent rays against the same node at once, using SIMD. If all rays in the packet miss a node, skip it. Ray packets exploit the fact that primary rays from adjacent pixels are nearly parallel and hit similar parts of the BVH.

The widget below fires rays through a 2D BVH built by median split. A click casts a ray from the middle of the left edge toward the click, and the traversal animates one popped node at a time: green boxes are hits, red boxes with X marks are misses, and amber dashed boxes were hit by the ray but start beyond the closest hit already found, so ordered traversal skips them.
Live · Ray-BVH traversal
new scene
nodes popped
0
entered
0
early-outs
0
Objects, the BVH boxes, and a ray, with readouts for nodes popped, nodes entered and early-outs. Click (or tap) anywhere on the canvas to cast a ray at that point. Each popped node's AABB goes through the slab test. A miss discards the node and its whole subtree; a node the ray does hit is still skipped if its entry distance lies beyond the closest object hit so far. "Early-outs" counts both kinds, with the number of internal nodes (whole subtrees) in parentheses. The closest object hit so far is filled yellow.
