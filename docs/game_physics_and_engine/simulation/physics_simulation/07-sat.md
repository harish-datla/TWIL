# SAT (Separating Axis Theorem)

The SAT is the other major narrow-phase algorithm, and the one that also says which way to push two overlapping boxes apart and how far. For convex polytopes, it is often faster than GJK for simple shapes (boxes, triangles) because it directly produces the contact normal and penetration depth without a separate EPA pass.

The theorem: two convex shapes are separated if and only if there exists a line (an "axis") on which their projections do not overlap. For polygons, the candidate axes are the face normals of both shapes. For polyhedra in 3D, add the cross products of every edge pair between the two shapes. If all candidate axes show overlap, the shapes are colliding, and the axis with minimum overlap is the contact normal.

Worked example

Two unrotated boxes: A spans x = 0 to 2 and y = 0 to 2, B spans x = 1.5 to 4 and y = 1 to 3. On the x axis their projections overlap by 2 − 1.5 = 0.5; on the y axis by 2 − 1 = 1. Both axes overlap, so the boxes collide, and the smaller overlap picks the answer: push B along +x by 0.5.
Live · SAT separating-axis demo
rotate shape B 0°
axes tested
2 of 2 (stopped)
overlap
none (separated)
contact normal
n/a
Two boxes, their projections onto each candidate axis as blue and yellow bars, and readouts for the axes tested, the overlap and the contact normal. Drag either box and rotate B with the slider. Each dashed line is a candidate axis (a face normal) drawn through the middle of the canvas, and the blue and yellow bars on it are the two boxes' projections onto that axis. Two boxes need at most four axes in 2D, two per box, and only two when they share an orientation. If any pair of bars is disjoint, that axis separates the boxes, turns red, and the test stops there. When every pair overlaps, the axis with the smallest overlap (yellow) is the contact normal, pointing from A to B, and that overlap is the penetration depth. The red patch is the region where the boxes intersect; a manifold builder clips edges to find contact points inside it.
