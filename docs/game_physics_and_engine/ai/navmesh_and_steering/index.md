# Game AI: Navmesh & Steering

Pathfinding needs more than a route: an agent needs a walkable map, a taut path through it, and movement that handles acceleration, flocking, and nearby agents. This tutorial builds navmeshes, funnel paths, steering behaviors, and local avoidance.

Time: ~55 min  
Level: Senior  
Prerequisites: Graph Traversal and Behavior Trees.  
Phase: 11 · AI

## What you’ll be able to do

- Explain when a navmesh beats a grid and how its polygon graph works.
- Read Recast build settings and reason about agent-radius limits.
- Turn an A* polygon corridor into a taut path with the funnel algorithm.
- Write and blend seek, arrive, wander, flocking, and steering behaviors.
- Choose between VO, RVO, and ORCA local avoidance.

## Sections

- [Why Not a Grid](01-why-not-a-grid.md)
- [The Navigation Mesh](02-the-navigation-mesh.md)
- [Generating It](03-generating-it.md)
- [Corridor to Taut Path](04-corridor-to-taut-path.md)
- [Path to Motion: Steering](05-path-to-motion-steering.md)
- [Combining & Flocking](06-combining-and-flocking.md)
- [Local Avoidance](07-local-avoidance.md)
- [Crowds & the BT Seam](08-crowds-and-the-bt-seam.md)
- [Pitfalls](09-pitfalls.md)
- [What’s Next](10-whats-next.md)
