# Partitioning into cells

Streaming needs a unit: something small enough to load in the background and large enough that the loader isn't juggling millions of separate objects. Cut the world into a grid of cells (sometimes called sectors or tiles): each cell owns the actors, meshes, and collision in its footprint, baked as one loadable unit. The cell the player stands in plus a radius of neighbors is what stays resident. Unreal's World Partition does this with a 2D runtime spatial hash grid by default, which removed the need to split large maps into hand-managed sublevels[2]. The grid-versus-quadtree mechanics are the Spatial Partitioning tutorial's territory; here the only new state per cell is its residency.
Cell size is a churn-versus-spike tradeoff

Cell size has to be small enough that loading one doesn't spike memory or the frame, but large enough that cells don't churn (load and unload rapidly) as the player moves near a boundary[3]. There is no universal number. Epic's World Partition documentation illustrates the settings with 256 m cells and a 768 m loading range[2], and the StraySpark guide suggests anything from 64 m to 256 m depending on density[3]. A regular grid is cheap to build and extends in any direction, but wastes resolution where the world is sparse; a quadtree adapts to uneven density at a higher update cost. Grids are the common choice; the right one depends on how evenly the world's content is spread.

Worked example

Take Epic's illustration: 256 m cells and a 768 m loading range, 3 cells of radius. A player riding at 30 m/s (108 km/h) crosses a cell every 256 / 30 ≈ 8.5 s, and each crossing brings roughly one new row of cells across the circle's leading edge, about 2 × 768 / 256 = 6 of them. That's around 6 cells every 8.5 s, 0.7 cells a second, that the loader has to sustain. Halve the cell size and the rows come twice as often with twice as many cells each, but each cell is a quarter of the size: the same data per second, cut into smaller, more frequent loads.

The map below streams cells around a wandering player. It starts with the cells around the player resident, as if a loading screen had filled them. Cells inside the load radius queue, then load over real time; cells past the eviction radius are freed. Drop the bandwidth or raise the speed until the player outruns the loader and the player dot turns red: that's pop-in:
Load radius 3
Bandwidth (cells/s) 6
Player speed (cells/s) 1.1
velocity prediction
Watch the ring of cells follow the player, then drag Bandwidth down to 3 and Player speed to the top. Amber = queued, blue = loading (filling), green = resident; the dashed ring is the load radius, and cells evict at a larger radius (hysteresis, §6). The loader works through the queued cells nearest the ring's center first, at the rate the bandwidth slider sets. Outrun it (raise the speed or drop the bandwidth) and the player enters a cell that hasn't arrived: the dot turns red and the pop-in counter ticks. At full speed with a bandwidth of 3, prediction holds pop-in to roughly 30 to 35 events a minute, against about 110 without it. Prediction centers the ring ahead of travel, so when the player bounces off the edge of the map, cells that were loading ahead of the old heading end up behind the new one.
