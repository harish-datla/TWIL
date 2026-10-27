# Semi-Fixed: Clamp the Step


The spring blew up because one step was too big, so the first partial fix is to never integrate more than a maximum dt at once. If a frame took longer, take several capped steps[3].
Semi-fixed timestep (Fiedler)
C++
Rust

while (frameTime > 0.0) {
    double step = std::min(frameTime, dt);  // never exceed dt
    integrate(state, step);
    frameTime -= step;
}

This bounds the step size, which removes large-step instability and reduces tunneling (a fast enough object still crosses a thin wall within one dt). It is still not deterministic: the last step each frame is a partial dt, so step sizes still vary frame to frame.

Worked example

With dt = 1/60 s (16.67 ms), a 40 ms frame runs steps of 16.67, 16.67 and 6.67 ms. A 35 ms frame on another machine runs 16.67, 16.67 and 1.67 ms. No step is too big any more, but the two machines took different step sequences, so their results drift apart just as in the falling-ball example.

That trailing remainder is exactly what the accumulator pattern fixes, by deferring it instead of stepping it.

[Back to The Game Loop & Time](index.md)
