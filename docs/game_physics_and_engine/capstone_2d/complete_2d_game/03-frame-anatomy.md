# Frame Anatomy

Every display frame follows the same order: poll OS events into an input snapshot, consume elapsed time with zero or more fixed simulation steps, then render once from an interpolated snapshot.

```text
poll input → fixed simulate(dt) steps → interpolate → render
```

The accumulator stores real time not yet simulated. For leftover `alpha = accumulator / dt`:

```text
renderPosition = previous + (current − previous) × alpha
```

At 144 Hz, frames add 6.94 ms while the simulation advances in 16.67 ms steps. After three display frames, one simulation step has run and 4.17 ms remains, giving `alpha = 0.25`. Interpolation makes the ship glide at display rate while the simulation remains deterministic.

Render only; never mutate the world during rendering. Raw current state stutters at the simulation rate; extrapolation can overshoot collisions and snap back. Interpolation adds up to one step of latency but blends already-computed states safely. Clamp large frame times and cap catch-up steps: if simulation costs more than the time it simulates, unbounded catch-up becomes the spiral of death. A guard may slow simulation relative to real time, but prevents frame time from growing without bound.
