# The Game Loop & Time

Your game runs in slow motion on a weak laptop and too fast on a 144 Hz monitor. You scale movement by frame time to fix it, and now a one-frame hitch throws a crate through a wall. Both bugs live in the few lines of the game loop. This page builds that loop from the naive version up, in C++ and Rust, stopping at every failure that forced the next change, and ends with a fixed timestep (the simulation always advances by the same small step) driven by an accumulator (a running total of real time not yet simulated), with a render that blends between simulation states.
Time
~65 min
Level
Beginner to mid
Prereqs
You can read C++ or Rust. The 3D Math tutorial helps, since interpolation blends two states.
Stack
C++ & Rust
◂ Build a Game Engine
Phase 3 · The runtime spine
Next · Platform & Window ▸
What you'll be able to do by the end

    Explain why a naive loop runs at different speeds on different machines, and why scaling by frame time brings new bugs.
    Write a fixed-timestep loop with an accumulator in C++ or Rust, and work out how many steps it runs on any given frame.
    Render smoothly between simulation ticks with interpolation, and say when extrapolation's overshoot is acceptable.
    Guard a loop against the spiral of death, and know what each guard costs.
    Pick a monotonic clock, pace frames without trusting Sleep, and list what a fixed step doesn't make deterministic.


## Sections

- [The Loop and the Kinds of Time](01-the-loop-and-the-kinds-of-time.md)
- [The Naive Loop](02-the-naive-loop.md)
- [Variable Timestep](03-variable-timestep.md)
- [Semi-Fixed: Clamp the Step](04-semi-fixed-clamp-the-step.md)
- [Fixed Timestep and the Accumulator](05-fixed-timestep-and-the-accumulator.md)
- [Decoupling Sim from Render](06-decoupling-sim-from-render.md)
- [Interpolation, Not Extrapolation](07-interpolation-not-extrapolation.md)
- [The Spiral of Death](08-the-spiral-of-death.md)
- [Determinism](09-determinism.md)
- [Timing Sources](10-timing-sources.md)
- [Case Studies](11-case-studies.md)
- [Pitfalls](12-pitfalls.md)
- [What's Next](13-whats-next.md)
