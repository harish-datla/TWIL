# Why simulate physics

An artist places a tower of crates in a level, and when a grenade goes off they need to topple, bounce and settle instead of hanging in the air. A physics engine is the runtime that turns artist-placed geometry into something that responds to forces: crates that stack, ragdolls that collapse, vehicles that corner, destruction that propagates. The simulation usually runs at a fixed rate (commonly 50 or 60 Hz for gameplay, higher for vehicle dynamics and VR hand interaction) inside an outer variable-rate render loop. Every frame the engine (1) integrates forces into velocities and positions, (2) detects collisions between all pairs of overlapping shapes, (3) generates contact points where shapes touch, (4) solves a system of constraints that prevent penetration and enforce joints, and (5) writes the resulting transforms back for rendering.

The workload is dominated by step 4, because the solver iterates over every constraint several times per frame. Scale to a destruction scene with thousands of fragments and the constraint solver becomes the frame-rate limiter.

Worked example

A stack of 20 boxes has 20 face-to-face contacts (19 between boxes, 1 with the ground), and with 4 contact points each that's roughly 80 contact constraints. At 8 solver iterations and 120 Hz, that's 80 × 8 × 120 = 76,800 constraint solves per second for a single stack.

The core loop is small. A minimal rigid body engine for convex shapes is on the order of a few thousand lines of C++. Most of those lines are in collision detection and constraint solving; integration is about 10 lines. Most of the engineering effort goes into stability, determinism, and performance at scale.
