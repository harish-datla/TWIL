# Morton Order and Swizzling


A texture is stored row by row, so the texel directly below the one you just read is a whole row away in memory, likely on another cache line. Bilinear filtering reads a 2×2 block every time, so half of every lookup lands far from the other half. You want a memory order where points close together in 2D are also close together in 1D.

Morton order gives you that. Packing puts independent fields side by side; Morton order interleaves the bits of two coordinates instead. The encode is pure shift-and-mask: spread x into the even bit positions, spread y into the odd positions, and OR them together. Guy Morton described it in a 1966 IBM report on ordering the records of a geodetic database[6].

Worked example

Encode x = 5 (101), y = 3 (011). Write the bits from the top as y2 x2 y1 x1 y0 x0: 0 1 1 0 1 1, which is 0b011011 = 27.

Now a 2×2 block: (2, 2), (3, 2), (2, 3), (3, 3). Their Morton indices are 12, 13, 14, 15, four in a row. Their row-major indices on an 8-wide grid (y × 8 + x) are 18, 19, 26 and 27: two pairs, a row apart.

Walking the indices 0, 1, 2, 3 traces a Z shape, and the Zs repeat at every scale (hence "Z-order curve"). That keeps spatial neighbors mostly near each other in memory. GPUs commonly store textures in tiled or swizzled layouts, Morton order or close relatives of it, so that a 2D sampling neighborhood tends to share cache lines instead of spanning several rows. Linear octrees and quadtrees store nodes sorted by Morton code, computing parent and child relationships with shifts instead of pointer chases (the spatial partitioning tutorial builds those trees). Parallel BVH builders in the LBVH family sort primitives by Morton code as their first step.
Live · Morton (Z-order) tracer
Z-order
row-major
❚❚ pause

index
    25 (011001)
decoded
    x=5 (101) y=2 (010)
interleave
    011001

blue = x bits (even positions) · red = y bits (odd positions)
The path visits the 64 cells of an 8×8 grid in index order, and the readout decodes the current index into x and y. The interleave line colors each index bit by the coordinate it came from: they alternate y, x, y, x reading from the top. Switch to row-major and the path makes long horizontal sweeps: cell 7 and cell 8 are neighbors in index but at opposite ends of the grid. Z-order keeps most jumps small, which is what cache locality needs. Its big diagonal leaps are where the curve crosses into the next quadrant, the curve's known weak spot: (3, 3) is index 15, but its right-hand neighbor (4, 3) is 26.

For wider coordinates you do not loop bit by bit. The standard trick is a sequence of shift-and-mask steps with "magic" constants that spread the bits in logarithmically few operations, the same lane-by-lane style as the SWAR popcount in §15[1]. On x86 with BMI2, one PDEP per coordinate deposits its bits into an interleave mask, so a 2D encode is two PDEPs and an OR (§16).

[Back to Bit Shifting from Scratch](index.md)
