# Integration methods

A crate is in the air, gravity pulls on it, and the engine has 1/60 s to say where it is next frame. Integration is the step that turns forces into motion. Given a body with mass m, position x, velocity v, and net force F, Newton's second law gives the acceleration, how fast the velocity changes:
NEWTON II F = ma  →  a = Fm

In words: a body's acceleration is the total force on it divided by its mass, so the same push moves a light crate faster than a heavy one. Hover any symbol to see what it stands for.

The question is how to advance v and x by a discrete timestep dt. Four methods cover the space:
Method	Update rule	Stability	Game use
Explicit Euler 	x += v*dt, v += a*dt 	Unstable. Gains energy on oscillatory systems, diverges on springs. 	Almost never. A common beginner mistake.
Semi-implicit Euler 	v += a*dt, x += v*dt (velocity first) 	Symplectic. No secular energy drift; energy oscillates around the true value while the step is small enough (below 2/ω for a spring). First order, with some phase error. 	The usual choice in game engines (Box2D, Bullet, Jolt, PhysX); Catto's integration talk recommends it[11].
Velocity Verlet 	x += v*dt + 0.5*a*dt2; compute new a; v += 0.5*(a_old + a_new)*dt 	Symplectic, second-order accurate. Slightly better phase accuracy than semi-implicit Euler. 	Molecular dynamics. Rare in impulse-based rigid-body engines, whose contact solvers work directly on velocities and whose forces often depend on velocity (damping, friction), which the scheme assumes away.
RK4 	Four force evaluations per step, weighted average. 	Fourth-order accurate but not symplectic. Exhibits secular energy drift on long simulations. 	Offline and scientific simulation. Rare in games: each of the four stages would need its own collision and constraint pass, and Catto's advice is that first-order accuracy is usually sufficient and RK4 can be ignored[11].

Order of operations matters. Explicit Euler updates position with the old velocity, then updates velocity. Semi-implicit Euler updates velocity first, then uses the new velocity to update position. That single reorder makes the integrator symplectic: among other things it preserves phase-space area, and for a conservative system it keeps the energy bounded instead of pumping it into oscillations. The widget below makes the difference visible.

Worked example

Take the widget's defaults: a unit mass on a spring of stiffness k = 20, stepped at dt = 0.03 s. Explicit Euler multiplies the spring's energy by 1 + k·dt² = 1 + 20 × 0.0009 = 1.018 every step. That looks harmless, but after 100 steps (3 s) the energy has grown 1.018100 = 5.95 times. Semi-implicit Euler's energy only wobbles around the true value, as long as dt stays below 2 ÷ √k = 0.45 s.
Live · Integration method comparison (mass on spring)
play
reset
timestep dt 0.030 s
spring stiffness k (mass 1) 20
Explicit Euler E / E₀
1.000
Semi-implicit E / E₀
1.000
Verlet E / E₀
1.000
Three panels, one per integrator, with E / E₀ readouts below. Press play, then raise dt and k. Each panel plots phase space: position across, velocity divided by √k up, so the exact motion is a circle of radius 1 and E / E₀ is its energy relative to the start. Explicit Euler advances position with the old velocity and velocity with the force at the old position; for this spring (mass 1) every step multiplies the energy by exactly 1 + k·dt², so the orbit spirals outward and E / E₀ climbs without bound. Semi-implicit Euler moves position with the new velocity; its orbit is a slightly tilted ellipse that closes on itself, so E / E₀ wobbles near 1 without drifting. Velocity Verlet is second order and stays closer to the circle. Raise dt or k and explicit Euler diverges faster, since the growth factor is 1 + k·dt².
Check Your Understanding Q1 of 1

Your physics simulation uses explicit Euler and objects on springs gain energy every frame, eventually flying off screen. Which fix is correct?
Add damping to every spring to counteract the energy gain.
Switch to semi-implicit Euler: update velocity before position.
Halve the timestep. Smaller dt means less energy gain per step.
0/1 correct
