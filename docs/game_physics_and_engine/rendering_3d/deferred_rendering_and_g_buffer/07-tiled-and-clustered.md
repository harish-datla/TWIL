# Tiled and Clustered

Even deferred lighting wastes work if every pixel loops over every light. Tiled shading divides the screen into 2D tiles and builds, usually in compute, a list of lights overlapping each tile. A pixel loops only its tile’s list. Clustered shading adds depth slices, giving each tile-and-depth box its own list.

Tiled bounds a tile by its nearest and farthest surface. A near railing and far wall can therefore assign every light between them to every pixel. Clusters separate those surfaces so each pixel sees only lights in its depth slice.

Use exponential depth boundaries:

```text
Z_k = near × (far / near)^(k / N)
```

This makes each slice a fixed percentage deeper than the previous one. With near 0.1 m, far 100 m, and 16 slices, the per-slice ratio is about 1.54. Clusters still over-approximate the exact pixel and can include lights that touch only a corner, but they greatly reduce irrelevant lights.

Tiled and clustered culling are orthogonal to deferred versus forward: tiled deferred, clustered deferred, and clustered forward all exist.
