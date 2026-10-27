# Constraint-based dynamics

A crate sits on a table, a door swings on a hinge, a wrecking ball hangs from a chain: each is a rule about how bodies may move relative to each other. A constraint is a restriction on how bodies may move. "These two bodies must not interpenetrate" is a contact constraint. "This body is attached to that body at a hinge" is a joint constraint. The constraint solver's job: find impulses that satisfy all active constraints simultaneously.

The formal framework (Baraff 1994[3], Catto 2005[5], Catto 2014[8]): each constraint is a scalar function C of the positions and orientations of the involved bodies, zero when the rule holds. Differentiating it gives the rule at the velocity level, which is the form the solver works with:
CONSTRAINT C(x) = 0  ⇒  dCdt = Jv = 0  →  solved as   Jv + b = 0

In words: if the rule holds, its rate of change must stay zero too, so the bodies' velocities must not push it out of shape; the solver aims for that with a small bias term added. Differentiating C gives the velocity constraint; the solver enforces it with an added bias term b that corrects drift. The J matrix (the Jacobian) maps body velocities to the rate of constraint violation. The solver computes an impulse lambda along each constraint's Jacobian direction such that the velocity-level constraint is satisfied.

Position drift (bodies slowly sinking into each other) is corrected by Baumgarte stabilization: add a bias term to the velocity constraint that is proportional to the current position error. For a contact, C is the signed separation along the normal, negative while the bodies overlap, so b comes out negative and J v = −b asks for a separating velocity:
BAUMGARTE b = betadt · C

In words: ask for a velocity that would close a fixed fraction, beta, of the current error in one step.

Worked example

A crate has sunk 2 cm into the floor, so C = −0.02 m. With beta = 0.2 and dt = 1/60 s, b = 0.2 × 60 × (−0.02) = −0.24 m/s, so the solver asks for a 0.24 m/s separating speed. That moves the crate 0.24 ÷ 60 = 4 mm this step, a fifth of the error, and the rest closes over the next few steps.
