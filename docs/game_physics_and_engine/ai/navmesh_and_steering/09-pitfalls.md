# Pitfalls
"Grids are obsolete"
Tile games win with grids. Navmesh wins on compactness + free-form paths.
Pathing through poly centers
Zig-zag. Run the funnel over the corridor (O(n), not a search).
Recast = Detour
Recast builds the mesh; Detour queries it. Different libraries.
Conflating pathfinding & steering
Global/discrete vs local/continuous. Steering runs on top of a path.
"Arrive is seek with low max speed"
Arrive brakes in a slowing radius; seek overshoots and loops back.
Swapping boids rules
Separation=position, cohesion=centroid, alignment=heading. Don't mix them.
"RVO removes oscillation"
It removes plain VO's two-agent oscillation; with three or more agents it can come back.
Avoidance as navigation
It gets stuck in concave geometry. Always run a global path under it.
