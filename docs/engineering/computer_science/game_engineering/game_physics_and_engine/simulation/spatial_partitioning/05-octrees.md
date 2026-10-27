# Octrees

A 3D world adds height: a flying drone, a multi-story building, a cave under a mountain. An octree is a quadtree in three dimensions. Each node splits into eight children (2³). Insert, remove, and query follow the same logic, with an extra axis.

Two storage layouts are common. A pointer-based octree stores eight child pointers per node, same as the quadtree above. A linear octree encodes each node's position as a Morton code and stores all nodes in a flat sorted array. The Morton code (the index along a Z-order curve) interleaves the bits of the x, y, z cell coordinates. A parent's code is its child's code shifted right by three bits, and neighbors are found with bit arithmetic on the interleaved coordinates, so no pointer is ever dereferenced.

Worked example

Cell (x, y, z) = (5, 3, 1) has coordinate bits x = 101, y = 011, z = 001. Interleaving them one bit at a time, z y x from the lowest bit up, gives 001 010 111 in binary, Morton code 87. Shift right by three bits and you get 001 010, which is 10: exactly the Morton code of the parent cell (2, 1, 0), each coordinate halved.

Linear layouts matter for GPU work, and three GPU structures that are often lumped together use them differently. Morton-ordered linear hierarchies are the LBVH line: Lauterbach et al. sort primitives by Morton code to build a hierarchy in parallel[10], and Karras builds every level at once from a binary radix tree over those codes[11]. Sparse voxel octrees are a different design: Laine and Karras encode topology with 64-bit child descriptors carrying a valid mask, a leaf mask, and a relative child pointer, not with Morton codes[12]. NVIDIA's GVDB takes a third route: the VDB topology, a sparse hierarchy of grids whose branching factor is configurable per level, indexing into a voxel atlas and built for volumes whose topology changes every frame. An octree is the special case with a branching factor of two per axis, and GVDB's programming guide notes that such a tree "quickly requires more than 5 levels", its default depth limit, so the usual configurations branch much wider[13].

On the CPU, engine octrees are usually depth-limited and stored either with child pointers or in flat arrays addressed by index. Unreal's generic octree, TOctree2, is the second kind: its nodes live in a TArray with parent links and a free list, which is cheaper to update in place than a web of heap-allocated nodes[25].
