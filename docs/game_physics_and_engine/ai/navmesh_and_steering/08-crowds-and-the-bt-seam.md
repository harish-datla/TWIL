# Crowds & the BT seam

Thirty guards in one courtyard all want the same doorway. Steering each one in isolation gives a pile-up; something has to plan their corridors, spacing and avoidance together. A crowd manager runs many agents together: path-corridor following, local avoidance, radius-based separation, and replanning. Detour's dtCrowd is the open-source reference. Its avoidance scores sampled candidate velocities against RVO-style velocity obstacles rather than solving ORCA's LP, and its documentation puts the practical maximum at 20 to 30 agents under one crowd manager, which takes over their positions entirely[9]. For hundreds to thousands of units, one documented approach is flow-field tiles (a shared vector field per goal, shipped in Supreme Commander 2) in place of per-agent pathfinding[10].

Worked example

A flow field for a 100 × 100 tile region stores 10,000 direction vectors, one per tile, pointing along the best route to the goal. Ten units or a thousand read the same field, so its cost depends on the area it covers, while per-agent pathfinding runs one search per unit: a thousand searches for a thousand units.
The decision / navigation / physics split

This module implements the NavAgent the behavior tree's MoveTo leaf already calls: requestPath(target) (A* → corridor → funnel → path) and hasArrived(), from which the leaf returns RUNNING, SUCCESS, or FAILURE. The BT decides where, navigation gets there, and steering hands a desired velocity to the physics character controller, which owns the final position (collision, slopes, step-up). Steering writes velocity, never position; that is the contract applySteering in §5 follows. Agent radius is load-bearing twice: it insets the navmesh and sets the avoidance spacing. Dynamic obstacles need either a navmesh change (for obstacles that move and then stay put, such as a crate pushed into a doorway: rebuild the affected tiles or carve a hole) or runtime avoidance (for obstacles that keep moving). Unity's navigation documentation draws the line the same way[11].
Check Your UnderstandingQ1 of 2

A* on your navmesh returns a path, but the agent walks a long zig-zag instead of a straight line across an open area. Why, and the fix?
The path is routed through polygon centers; run the funnel (string-pulling) over the A* corridor to pull a taut path: one sweep over the corridor, not another search.
The A* heuristic is inadmissible; switch to Dijkstra.
The navmesh polygons are convex; make them concave so paths can curve.
0/2 correct

Wrong answers, and why: a zig-zag is a corridor-extraction problem fixed by the funnel (not the heuristic or polygon convexity); and agents stuck in concave geometry need a global path (local avoidance is never a planner; reciprocity and horizon don't add a map).
