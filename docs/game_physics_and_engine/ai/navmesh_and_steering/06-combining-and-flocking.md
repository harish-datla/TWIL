# Combining & flocking

A sheep in a flock wants to stay near the others, not bump into them, and head the same way, all in the same frame. So each agent runs several steering behaviors at once, and their outputs have to become one acceleration. Multiple behaviors blend by weighted sum (add each steering vector × weight, truncate), priority arbitration (take the highest-priority non-trivial behavior), or context steering (each behavior writes interest and danger into directional bins, and the agent picks the best bin)[13]. Flocking (Reynolds' boids) is three behaviors over local neighbors[5]: separation (steer away from neighbors' positions), cohesion (steer toward their centroid), alignment (match their heading).

Worked example

An agent seeks a doorway to its right, steering (+1, 0) at weight 1, while separation pushes it away from a neighbor standing in the doorway, (−1, 0) at weight 1. The weighted sum is (0, 0): the agent stands still, and stays there as long as the neighbor does. Priority arbitration would let one behavior win outright; context steering would mark the doorway direction as both interesting and dangerous and pick the best remaining direction.
Weighted sum can cancel, and separation is not collision-free

Naive weighted sum has two failure modes: equal-and-opposite cancellation (two behaviors point opposite and sum to about zero, so the agent stalls) and getting stuck in local minima (wedged against a concave obstacle). Priority arbitration and context steering exist to avoid these, each with its own cost (priority arbitration can jitter when the winning behavior flips). Boids is purely local: the flock pattern emerges with no central controller, and the three rules are distinct: separation = repel from positions, cohesion = attract to the centroid, alignment = match heading. Separation gives soft spacing, not guaranteed collision-free motion; that is local avoidance's job (§7).
