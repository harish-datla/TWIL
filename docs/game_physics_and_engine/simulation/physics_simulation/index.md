# Physics Simulation

This tutorial builds a real-time rigid-body physics runtime from integration through collision detection, contact manifolds, constraint solving, fixed timesteps, and continuous collision detection.

Time: ~60 min  
Level: Mid to senior  
Prerequisites: Vectors, dot/cross product, basic calculus, and C++ or Rust.  
Phase: 9 · Simulation

## What you’ll be able to do

- Integrate rigid bodies with semi-implicit Euler and explain its stability.
- Prune collision pairs with sweep and prune, then test convex shapes with GJK and SAT.
- Build contact manifolds and solve them with sequential impulses, warm starting, and Baumgarte correction.
- Run a fixed-timestep simulation with an accumulator and render interpolation.
- Stop tunneling with swept CCD or speculative contacts.

## Sections

- [Why Simulate Physics](01-why-simulate-physics.md)
- [A Short History](02-a-short-history.md)
- [Integration Methods](03-integration-methods.md)
- [The Semi-Implicit Euler Step](04-the-semi-implicit-euler-step.md)
- [Collision Detection: Broad Phase](05-collision-detection-broad-phase.md)
- [Narrow Phase: GJK](06-narrow-phase-gjk.md)
- [SAT](07-sat.md)
- [Contact Points and Manifolds](08-contact-points-and-manifolds.md)
- [Constraint-Based Dynamics](09-constraint-based-dynamics.md)
- [Sequential Impulses](10-sequential-impulses.md)
- [Common Constraints](11-common-constraints.md)
- [Substeps and Fixed Timestep](12-substeps-and-fixed-timestep.md)
- [Continuous Collision Detection](13-continuous-collision-detection.md)
- [Case Studies](14-case-studies.md)
- [Pitfalls](15-pitfalls.md)
- [What’s Next](16-whats-next.md)
