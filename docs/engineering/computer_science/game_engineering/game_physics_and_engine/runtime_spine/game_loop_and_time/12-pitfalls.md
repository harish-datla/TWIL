# Pitfalls

Game speed tracks frame rate
No delta time, or motion not scaled by it. The naive loop.
Subtle stutter at a fixed step
Rendering the raw tick state without interpolating by the leftover accumulator.
Simulation runs slow
Discarding the accumulator remainder instead of carrying it.
Freeze under load
No frame-time clamp or step cap: the spiral of death. A cap that keeps the leftover time only delays it.
Objects trail or overshoot
Interpolation adds up to a tick of latency; extrapolation snaps on a bounce. Pick per system.
Negative or huge dt
Using a wall clock instead of a monotonic one; an NTP step or manual clock change.
Replays desync across machines
Fixed step alone is not determinism; float and toolchain differences remain.

[Back to The Game Loop & Time](index.md)
