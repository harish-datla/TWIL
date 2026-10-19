# Why not a grid

The simplest map of where an agent can walk is a grid: chop the floor into square cells, mark each one walkable or blocked, and search from cell to neighboring cell, with 4 or 8 neighbors per cell. It's the right structure for tile-based games (the world is a grid) and is simple to build and query. For continuous 3D worlds a navmesh, a set of walkable polygons covering the floor (§2), has three advantages, though grids remain the right choice when the game itself is made of tiles:

    Memory/resolution: grid cost scales with area at a fixed cell size; a big open room is one navmesh polygon but thousands of cells, and the cell size has to shrink to fit narrow gaps, which multiplies the cell count everywhere.
    The blocky path: an 8-connected grid path is limited to 8 directions, so off-axis routes come out as staircases that need post-smoothing. A navmesh corridor plus the funnel (§4) gives a path that heads straight for each corner at any angle.
    3D / multi-level / slopes: a 2D grid can't natively represent ramps, bridges over bridges, or overlapping floors; a navmesh is polygons in 3D space.

Worked example

An empty 20 m × 20 m room at 0.25 m cells is 80 × 80 = 6,400 grid cells; as a navmesh it's one rectangle. If a 0.6 m gap elsewhere forces 0.2 m cells, the same room becomes 100 × 100 = 10,000 cells. Paths suffer too: an 8-connected grid path 10 cells across and 4 down takes 4 diagonal steps and 6 straight ones, 4 × 1.414 + 6 ≈ 11.66 cell widths, against √(10² + 4²) ≈ 10.77 for the straight line. That's about 8% longer, with a visible kink.
