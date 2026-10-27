# IEEE-754 Floating Point

A character 20 km from the world origin jitters, a replay desyncs on a teammate's PC, and the audio drops out when a reverb tail fades. All three are float bugs. Most positions, velocities and shader inputs in an engine are 32-bit floats, and floats round in small, specific, reproducible ways. This page builds a float from its 32 bits up, in C++ and Rust, so you can predict how much precision a value has, compare floats safely, keep long sums accurate, and see those three bugs coming.
Time
~65 min
Level
Beginner to mid
Prereqs
You can read C++ or Rust. Bit Shifting helps; the layout is all bit fields.
Stack
C++ & Rust
◂ Build a Game Engine
Phase 2 · Math & numerics
Next · The Game Loop & Time ▸
What you'll be able to do by the end

    Decode a float's 32 bits by hand into its sign, exponent and significand, and spot a subnormal, an infinity or a NaN in a memory dump.
    Work out how much precision a value has at a given magnitude, and explain why a world 20 km across jitters.
    Compare floats with a tolerance that holds at the magnitudes your code sees, and write NaN checks a compiler flag won't delete.
    Keep a long sum accurate with Kahan summation, and name the flags that silently undo it.
    List the causes of cross-machine divergence in a lockstep game, and stop subnormals from stalling an audio thread.


## Sections

- [Why Floats](01-why-floats.md)
- [The Bit Layout](02-the-bit-layout.md)
- [Normals & Subnormals](03-normals-and-subnormals.md)
- [Signed Zero, Infinity, NaN](04-signed-zero-infinity-nan.md)
- [Rounding, and Why 0.1 Isn't Exact](05-rounding-and-why-01-isnt-exact.md)
- [Machine Epsilon and ULP](06-machine-epsilon-and-ulp.md)
- [Catastrophic Cancellation](07-catastrophic-cancellation.md)
- [Summing a Million Numbers](08-summing-a-million-numbers.md)
- [FMA, Order, and -ffast-math](09-fma-order-and-ffast-math.md)
- [Comparing Floats](10-comparing-floats.md)
- [Determinism across Machines](11-determinism-across-machines.md)
- [The Subnormal Cliff](12-the-subnormal-cliff.md)
- [Pitfalls](13-pitfalls.md)
- [What's Next](14-whats-next.md)
