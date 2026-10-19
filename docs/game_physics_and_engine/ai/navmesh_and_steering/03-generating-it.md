# Generating it

Nobody wants to place polygons by hand over a 2 km level, and level geometry is rarely clean: walls overlap, floors have gaps, props intersect. The common automatic approach (Mononen's Recast, which powers the navigation in Unity, Unreal, Godot, and O3DE according to its README) is a voxelization pipeline, run at bake time[2]. The stages below follow Recast's own sample build[2]; Unity's documentation describes its bake in the same terms: rasterize to voxels, extract walkable surfaces, build the mesh[11].

    Mark and rasterize: flag triangles too steep to walk on (the slope limit), then rasterize the collision triangles into a solid heightfield of voxel spans.
    Filter walkable spans: keep low obstacles the agent can step over (climb height), drop ledges higher than the climb height, and drop spans without head clearance (agent height).
    Build a compact heightfield, erode the walkable area by the agent radius, and build a distance field (distance to the nearest border).
    Region-segment the walkable area (watershed in Recast's sample; monotone or layer partitioning as faster alternatives).
    Trace and simplify contours around each region.
    Polygonize the contours into convex polygons (≤6 verts); a detail mesh adds height samples.

Worked example

Take an agent radius of 0.6 m. Step 3 erodes the walkable area by 0.6 m from every wall, so the agent's center can only go where its body fits. A 1.5 m doorway keeps a 1.5 − 2 × 0.6 = 0.3 m strip down the middle and stays open. A 1.0 m doorway would need −0.2 m, so it closes: the navmesh for this agent has no link through it, and A* routes around. A smaller agent gets a different navmesh in which that doorway is open.
Recast is the generator; Detour is the runtime; the result is quantized

Keep them distinct: Recast builds the mesh, Detour queries it (pathfind, crowd). Voxelization trades exactness for robustness: it runs on arbitrary triangle soup with no cleanup, but the navmesh edges are quantized to the cell size rather than sitting exactly on the source geometry. Hand-authored navmesh (a designer places polygons) gives exact control and is still used for fix-ups, but doesn't scale to large auto-built worlds.
