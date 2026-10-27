# Corridor to taut path

A* hands back a list of polygons, and the obvious way to walk it, through the middle of each doorway, makes the guard weave like it's lost. Run A*, the shortest-path search guided by an estimate of the distance left, on the polygon dual graph. Each step needs a cost and a heuristic: Detour measures cost between the midpoints of the shared edges the path crosses (polygon centers are the simpler alternative) and uses straight-line distance to the goal as the heuristic, which is admissible since no path is shorter than a straight line. The Graph Traversal tutorial covers A* and the admissibility theorem, so this page builds on it. The key point: A* returns a polygon corridor (an ordered strip of polygons), not a line.
Don't walk through polygon centers: that's the zig-zag bug

Connecting polygon centroids, or the midpoints of the shared edges A* scored, gives a kinked, needlessly long path. The fix is the funnel (string-pulling) algorithm: pull a string taut around the corridor's inner corners. Each shared edge is a portal with a left and right vertex; the funnel keeps an apex and two feet and walks the portals, narrowing the wedge until a portal vertex crosses the opposite side, which marks a path corner. It is a sweep over the corridor, not another search. A corridor is a triangulated simple polygon, and shortest paths inside one take linear time with the classic funnel method[12]; Mononen's Simple Stupid Funnel, below, instead restarts its scan from the new apex after each corner, which re-reads some portals but is much shorter to write[3]. The result is the shortest path constrained to the corridor A* chose; a different corridor could be shorter. It also pulls the path tight against corner vertices, which is safe when the navmesh was eroded by the agent radius at build time (as Recast's is); systems that share one mesh across agent sizes inset each portal by the radius instead.

Worked example

Take the widget below on a 690 × 388 px canvas. The start sits at (55, 194) and the goal at (634, 194), and the three gaps between them alternate high, low, high. Through the gap midpoints the path is about 803 px long. The funnel path bends only at the lower corner of the first and third gaps and the upper corner of the second, and it's about 612 px: 24% shorter, from the same corridor.

The navmesh below is four rectangular polygons joined through three gaps in the walls, the portals; the corridor A* returns is all four. Drag the start (blue, in the first polygon) and the goal (amber, in the last), switch between the two paths, and press walk:
portal midpoints
funnel path
walk
Drag the start and goal, switch between the two paths, and press walk to send an agent along the selected one. The portal-midpoint path goes through the middle of each gap, so it zig-zags farther than it needs to. The funnel path is taut: the shortest line through that corridor, bending only at the gap corners it must round. The two lengths are shown bottom left; with the default start and goal on a desktop-width canvas the funnel path is about a quarter shorter. Walk the agent to compare.
The Simple Stupid Funnel over the corridor's portals
C++
Rust

// Twice the signed area of triangle abc, with Mononen's sign convention: in a y-up
// frame it is positive when c lies to the RIGHT of the ray a->b (the negated 2D cross
// product). Portals give left/right as seen walking along the corridor. With the
// ordinary cross product the comparisons below would all flip.
float triArea2(Vec2 a, Vec2 b, Vec2 c) { return (c.x-a.x)*(b.y-a.y) - (b.x-a.x)*(c.y-a.y); }

// portals[0] is the start point (left == right == start); the last portal is the goal.
std::vector<Vec2> stringPull(const std::vector<Portal>& portals) {
    std::vector<Vec2> path;
    Vec2 apex = portals[0].left, left = portals[0].left, right = portals[0].right;
    int apexIndex = 0, leftIndex = 0, rightIndex = 0;
    path.push_back(apex);
    for (int i = 1; i < static_cast<int>(portals.size()); ++i) {
        Vec2 newLeft = portals[i].left, newRight = portals[i].right;
        if (triArea2(apex, right, newRight) <= 0) {          // right foot moves inward (or stays)
            if (apex == right || triArea2(apex, left, newRight) > 0) {
                right = newRight; rightIndex = i;                    // narrow the funnel
            } else {                                                 // right crossed over left:
                path.push_back(left);                                // the left foot is a corner
                apex = left; apexIndex = leftIndex;                  // restart the funnel there
                left = apex; right = apex; leftIndex = apexIndex; rightIndex = apexIndex;
                i = apexIndex;                                       // ++i resumes after the new apex
                continue;
            }
        }
        if (triArea2(apex, left, newLeft) >= 0) {            // left foot: the mirror image
            if (apex == left || triArea2(apex, right, newLeft) < 0) {
                left = newLeft; leftIndex = i;
            } else {
                path.push_back(right);
                apex = right; apexIndex = rightIndex;
                left = apex; right = apex; leftIndex = apexIndex; rightIndex = apexIndex;
                i = apexIndex;
                continue;
            }
        }
    }
    if (!(path.back() == portals.back().left))   // the goal can already be the last corner
        path.push_back(portals.back().left);
    return path;
}

