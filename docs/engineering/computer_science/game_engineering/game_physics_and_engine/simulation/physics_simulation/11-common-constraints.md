# Common constraints

A door needs a hinge, a rope needs a maximum length, and a piston needs a slider, and none of them needs new solver code. Every constraint is a Jacobian row. The solver does not care what physical meaning the constraint has; it computes impulses that zero the velocity-level violation. The common constraints in a game engine:

    Contact (non-penetration). With velocities ordered (v_A, ω_A, v_B, ω_B), J = [−n, −(r_A × n), n, r_B × n], where n is the contact normal pointing from A to B and r is the vector from each centre of mass to the contact point, so J v is the separating speed. The impulse must be non-negative (push only). The solver code above implements the linear half of this row.
    Friction. Tangent-plane constraint with impulse clamped to the Coulomb cone. The maximum friction impulse is mu times the accumulated normal impulse.
    Distance joint. Keeps two anchor points at a fixed distance. C = |p_B − p_A| − L, where L is the rest length, and J = [−d, −(r_A × d), d, r_B × d], where d is the unit vector from anchor A to anchor B.
    Revolute joint (hinge). Forces two anchor points to coincide: two rows in 2D. In 3D the shared point takes three rows, and two more angular rows remove rotation about every axis except the hinge's.
    Prismatic joint (slider). Allows motion along one axis, constrains all others. The Jacobian zeroes out the allowed-axis component.

