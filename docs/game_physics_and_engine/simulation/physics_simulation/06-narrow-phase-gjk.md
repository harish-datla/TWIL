# Narrow phase: GJK

Two crates' bounding boxes overlap, but the crates themselves might not touch. The narrow phase tests the actual geometry of a candidate pair. For convex shapes, the dominant algorithm is GJK (Gilbert, Johnson, Keerthi, 1988[2]).

The idea: two convex shapes overlap if and only if their Minkowski difference contains the origin. GJK searches the Minkowski difference without constructing it explicitly, using support functions to probe specific directions. It builds a simplex (up to a tetrahedron in 3D) and checks whether it contains the origin.

If GJK reports overlap, EPA (Expanding Polytope Algorithm) takes the final simplex and expands it to find the penetration depth and contact normal.

Worked example

Square A is centered at the origin and square B 2.5 units to its right, both 2 units wide. Their Minkowski difference A − B is a square 4 units wide centered at (−2.5, 0), so it spans x = −4.5 to −0.5 and misses the origin. GJK doesn't need to build it: searching toward +x, A's farthest point is at x = 1 and B's farthest point toward −x is at x = 1.5, so the support point is at x = 1 − 1.5 = −0.5. Even the farthest point in that direction falls short of the origin, which proves separation after one support query. Move B to 1.5 units away and the same query gives +0.5, so GJK keeps searching and finds the overlap.
Live · GJK visualizer
step
run
reset
GJK iteration
0
simplex size
0
status
ready
Two shapes on the left and their Minkowski difference on the right, with the simplex in yellow and the origin marked. Drag the shapes, then press step a few times or run. GJK builds a simplex on the Minkowski difference (right panel) one support point at a time. Each step picks a search direction, queries both support functions, computes the Minkowski-difference support point, and adds it to the simplex. If the simplex contains the origin, the shapes overlap (green). If the new support point did not pass the origin, the shapes are separated (red). In 2D an overlap can only be proved by a triangle, so this widget needs at least three support points to report one; a separation can be proved sooner.
GJK core loop
C++
Rust
Copy

// Minkowski-difference support: farthest point on A in dir d, minus
// farthest point on B in dir -d. Declared first so gjkIntersect can call it.
Vec2 minkowskiSupport(const Shape& a, const Shape& b, Vec2 d) {
    return a.support(d) - b.support(-d);
}

// GJK intersection test for two convex shapes in 2D.
// Returns true if the shapes overlap.
bool gjkIntersect(const Shape& shapeA, const Shape& shapeB) {
    // pick an initial search direction (arbitrary; toward B's center works)
    Vec2 direction = shapeB.center() - shapeA.center();
    if (direction.lengthSquared() < 1e-8f) direction = Vec2(1.0f, 0.0f);

    // first support point on the Minkowski difference
    Vec2 support = minkowskiSupport(shapeA, shapeB, direction);
    Simplex simplex;
    simplex.add(support);

    // new search direction: toward the origin from the support point
    direction = -support;

    for (int iter = 0; iter < 32; ++iter) {
        Vec2 newPoint = minkowskiSupport(shapeA, shapeB, direction);

        // if the new point did not pass the origin, shapes are separated
        if (dot(newPoint, direction) < 0.0f) return false;

        simplex.add(newPoint);

        // check if the simplex contains the origin; if so, update direction
        if (simplex.containsOrigin(direction)) return true;
        // 'containsOrigin' also prunes the simplex to the closest feature
        // and sets 'direction' toward the origin from that feature
    }
    return false;  // iteration cap: only degenerate or touching cases get here
}

Check Your Understanding Q1 of 1

In GJK, when does the algorithm terminate and report that two shapes are separated?
When the simplex has 3 vertices (2D) or 4 vertices (3D).
When a new support point does not pass the origin along the search direction.
When the Minkowski difference has been fully constructed.
0/1 correct
