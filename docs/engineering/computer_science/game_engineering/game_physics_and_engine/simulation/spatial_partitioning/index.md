# Spatial Partitioning

Spatial partitioning makes broad-phase collision, frustum culling, raycasting, range queries, and nearest-neighbor lookups sublinear by organizing objects in space. This tutorial covers grids, trees, BVHs, spatial hashing, and dynamic updates.

Time: ~60 min  
Level: Junior to mid  
Prerequisites: C++ or pseudocode, basic Big O, and AABBs.  
Phase: 9 · Simulation

## What you’ll be able to do

- Choose between grids, quadtrees, octrees, BVHs, and spatial hashes.
- Build quadtrees and encode octree cells with Morton codes.
- Build and traverse BVHs using the surface area heuristic.
- Keep broad phases efficient with fat AABBs, sweep and prune, and dynamic updates.
- Compare the structures used by Bullet, Unreal, PhysX, Jolt, Nanite, and RTX hardware.

## Sections

- [Why Partition Space](01-why-partition-space.md)
- [A Short History](02-a-short-history.md)
- [Uniform Grids](03-uniform-grids.md)
- [Quadtrees](04-quadtrees.md)
- [Octrees](05-octrees.md)
- [BSP Trees](06-bsp-trees.md)
- [Bounding Volume Hierarchies](07-bounding-volume-hierarchies.md)
- [Surface Area Heuristic](08-surface-area-heuristic.md)
- [BVH Traversal](09-bvh-traversal.md)
- [Spatial Hashing](10-spatial-hashing.md)
- [Loose Bounds and Margin](11-loose-bounds-and-margin.md)
- [Queries](12-queries.md)
- [Sweep and Prune](13-sweep-and-prune.md)
- [Dynamic Updates](14-dynamic-updates.md)
- [Case Studies](15-case-studies.md)
- [Pitfalls](16-pitfalls.md)
- [What’s Next](17-whats-next.md)
