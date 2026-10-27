# Uniform grids

A top-down shooter has a few hundred bullets and enemies, all about the same size, and needs every bullet-enemy hit each frame. The simplest spatial structure for that: divide the world into a fixed grid of equal-sized cells. Each cell holds a list of objects whose centers (or bounding volumes) overlap that cell. Insert is O(1): hash the object's position to a cell index, append. Query is O(1) in the number of cells checked: compute which cells the query region overlaps, iterate their contents.

The spatial hash function maps cell coordinates to a flat array index. The usual choice is ((cellX * 73856093) ^ (cellY * 19349663)) % tableSize; the outer parentheses matter, because in C, C++ and Rust % binds tighter than ^. Those two constants are the primes Teschner et al. published in 2003 for exactly this problem, along with 83492791 for the third dimension[3]. The paper is candid about where they came from: the authors call them "large prime numbers, in our case" those three, say they "have not systematically investigated the characteristics of hash functions", and report that hash quality matters less as the table grows. They are a working choice, not a derived optimum.

Worked example

Cell (3, 5) hashes to 3 × 73,856,093 = 221,568,279 XOR 5 × 19,349,663 = 96,748,315, which is 149,986,828, and that mod a 4,096-entry table is bucket 3,596. Two different cells can land in the same bucket; a query just tests the bucket's objects against the real cell bounds and discards strays.

Grids win when the objects are roughly uniform in size and roughly uniform in distribution. Particle systems are the canonical example: thousands of similar-sized particles spread over the simulation domain. 2D games where entities are all within a factor of two in bounding-box size (top-down shooters, roguelikes, RTS unit collision) also fit well.

The failure mode is the large-object problem. An object that spans many cells must be inserted into all of them, and every query that touches any of those cells finds it. A single large terrain chunk in a grid of small cells can appear in hundreds of cells, multiplying both insertion cost and query noise. Hierarchical grids (multiple grid resolutions, one per object-size class) mitigate this at the cost of multiple lookups per query.
When grids beat trees

For uniform distributions of similar-sized objects, a grid is usually faster than a tree. The cell index comes straight from the coordinates, so reaching a cell's contents is one or two dependent loads. A balanced BVH walks about log2 n levels to reach a leaf (ten for a thousand objects), and on a cache-cold path each level is a dependent load. Particle systems, 2D bullet-hell games, and SPH fluid sims typically use grids.
