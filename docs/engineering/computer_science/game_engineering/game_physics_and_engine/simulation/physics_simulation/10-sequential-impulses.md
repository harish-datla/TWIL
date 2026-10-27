# Sequential impulses

A stack has dozens of contacts, and fixing one (pushing the top box up) disturbs the next (the box below it). The sequential impulse solver (Catto 2005[5], 2006[10]) is a projected Gauss-Seidel iteration over the constraints. For each constraint:

    Compute the relative velocity at the contact point along the constraint direction: relVel = J * v.
    Compute the impulse magnitude: lambda = -(relVel + bias) * effectiveMass, where effectiveMass = 1 / (J M⁻¹ Jᵀ) is the mass the constraint direction sees.
    Clamp: for a contact, the accumulated impulse must be non-negative (you can push but not pull). This is accumulated impulse clamping, not per-iteration clamping.
    Apply the impulse to both bodies' velocities immediately.

Worked example

A 2 kg crate rests on static ground. Gravity has just given it 0.1635 m/s of downward speed this step, so the relative velocity along the contact normal is −0.1635 m/s. Against an immovable ground the effective mass is the crate's own 2 kg, so the impulse is −(−0.1635 + 0) × 2 = 0.327 N·s, exactly the momentum gravity added (2 × 9.81 ÷ 60). It's positive, so the clamp keeps it, and applying it stops the crate.

Warm starting reuses the previous frame's accumulated impulses as the initial guess. Because the configuration changes little between frames, the warm-started guess is close to the converged answer, and the solver needs far fewer iterations. In Catto's 2005 tests at 10 iterations, a stack of three boxes without the contact cache slid apart and fell, while a stack of ten boxes with it settled within a few seconds and stayed up[5]. Matching contacts from one frame to the next is what makes this possible, which is why persistent manifolds (§8) matter.
Live · Sequential impulse solver (box stack)
play
reset
solver iterations 8
Warm starting (reuse last step's impulses)
bodies
9
contacts
0
max penetration
0.00 units
A stack of eight boxes with contact points marked, and readouts for bodies, contacts and the deepest overlap. Press play, then move solver iterations between 1 and 50 with warm starting on and off. The solver is the linear-only one from the code below: eight boxes, one contact per touching pair, a 0.5-unit penetration slop. Max penetration is the deepest overlap between two bodies this step. With warm starting off, every step starts from zero impulses and the solver runs out of iterations before it finishes: after the stack settles, the deepest overlap is about 4.4 units at 1 iteration, 0.9 at 8 (the velocity count Box2D v2.4's manual suggests), and only reaches the slop around 30. With warm starting on, 4 iterations already hold it at about 0.3, because each step starts from the last step's answer. At 1 or 2 iterations with warm starting on, the stack bounces instead: the carried-over impulse, position-correction push included, is reapplied in full and there are too few passes to take the excess back out. Each iteration costs time linear in the number of contacts.
sequential impulse contact solver
C++
Rust
Copy

// Sequential impulse solver for contact constraints.
// Iterates 'iterations' times over all contacts, accumulating impulses.
void solveContacts(
    std::vector<ContactConstraint>& contacts,
    std::vector<RigidBody>& bodies,
    int iterations, float dt)
{
    // warm start: apply last frame's accumulated impulses
    for (auto& contact : contacts) {
        Vec2 impulse = contact.normal * contact.accumulatedNormalImpulse
                     + contact.tangent * contact.accumulatedTangentImpulse;
        bodies[contact.bodyA].linearVelocity -= impulse * bodies[contact.bodyA].inverseMass;
        bodies[contact.bodyB].linearVelocity += impulse * bodies[contact.bodyB].inverseMass;
    }

    for (int iter = 0; iter < iterations; ++iter) {
        for (auto& contact : contacts) {
            RigidBody& bodyA = bodies[contact.bodyA];
            RigidBody& bodyB = bodies[contact.bodyB];

            // relative velocity at the contact along the normal (A to B);
            // positive means separating
            Vec2 relVel = bodyB.linearVelocity - bodyA.linearVelocity;
            float normalVel = dot(relVel, contact.normal);

            // Baumgarte bias b = (beta / dt) * C, where C is the signed
            // separation (negative while overlapping). The 0.01 slop lets
            // resting contacts overlap slightly instead of jittering. b <= 0,
            // so the impulse below asks for a separating velocity of -b.
            float separation = -contact.penetration;
            float bias = (0.2f / dt) * std::min(0.0f, separation + 0.01f);

            // impulse that drives J v + b to zero
            float lambda = -(normalVel + bias) * contact.effectiveMassNormal;

            // accumulated clamping: total impulse must be non-negative
            float oldAccum = contact.accumulatedNormalImpulse;
            contact.accumulatedNormalImpulse = std::max(0.0f, oldAccum + lambda);
            lambda = contact.accumulatedNormalImpulse - oldAccum;

            // apply impulse to both bodies
            Vec2 impulse = contact.normal * lambda;
            bodyA.linearVelocity -= impulse * bodyA.inverseMass;
            bodyB.linearVelocity += impulse * bodyB.inverseMass;

            // --- friction (Coulomb approximation) ---
            relVel = bodyB.linearVelocity - bodyA.linearVelocity;
            float tangentVel = dot(relVel, contact.tangent);
            float frictionLambda = -tangentVel * contact.effectiveMassTangent;

            // clamp friction to Coulomb cone
            float maxFriction = contact.friction * contact.accumulatedNormalImpulse;
            float oldFric = contact.accumulatedTangentImpulse;
            contact.accumulatedTangentImpulse = std::clamp(
                oldFric + frictionLambda, -maxFriction, maxFriction);
            frictionLambda = contact.accumulatedTangentImpulse - oldFric;

            Vec2 frictionImpulse = contact.tangent * frictionLambda;
            bodyA.linearVelocity -= frictionImpulse * bodyA.inverseMass;
            bodyB.linearVelocity += frictionImpulse * bodyB.inverseMass;
        }
    }
}

What's intentionally missing

The solver above is written to be read. A full one adds:

    Angular terms. Each contact impulse also applies a torque through the moment arm (contact point minus centre of mass), and the effective mass includes the inverse inertia tensor.
    Restitution. A bounce adds a velocity target of −e times the approach speed along the normal, captured before the iterations start.
    Contact matching. The warm start only helps if each contact's accumulated impulses survive from last step, which needs the feature-ID or proximity matching from §8.
    Sleeping and islands. Bodies at rest skip the solver, and connected groups of awake bodies are solved independently (and in parallel).
    Separate position correction. Baumgarte bias adds energy to the velocities it corrects; engines often correct penetration in a separate position pass (Jolt runs two by default) or with soft constraints instead.

Check Your Understanding Q1 of 1

A stack of boxes is jittery at 8 solver iterations. Your colleague suggests increasing to 64 iterations. What is the trade-off?
64 iterations is always better. More iterations means a more accurate solution with no downsides.
More iterations improve convergence but cost linear time per iteration. Warm starting provides more stability gain per unit cost than extra iterations.
The solver converges in exactly n iterations for n constraints, so 64 is the right number for a stack with 64 contacts.
0/1 correct
