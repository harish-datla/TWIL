# Contact points and manifolds

A crate resting flat on the floor touches it along a whole face; push on it with one contact point at the center and it rocks and spins on that point. A single collision between two boxes produces a contact manifold: a set of contact points, each with a position, normal, and penetration depth. Face-to-face contact between two boxes produces the intersection polygon of the two faces, up to eight vertices for two rectangles, which engines usually reduce to four points. Edge-to-edge generates 1 point.

Stable stacking requires persistent contact manifolds. The solver needs contacts to survive across frames so it can warm-start from the previous frame's solution. Contact point matching (by feature ID or by proximity) keeps the manifold stable as bodies shift slightly. Without persistence, every frame starts from scratch, and the solver needs many more iterations to converge.

Contact point reduction matters for performance. A face-to-face contact in 3D can produce an arbitrary polygon of contact points. Reducing it to 4 points, typically the deepest point plus points chosen to span the largest area in the contact plane, keeps the solver's constraint count bounded without losing stability.
