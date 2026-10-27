# Fixed Timestep and the Accumulator


To get the same simulation on every machine, every step has to be the same size, even though frames aren't. The canonical loop does it: accumulate real frame time, then consume it in whole dt chunks, carrying the leftover forward to the next frame[3]. Every simulation step is now exactly the same size, which makes it stable and reproducible.
Fixed timestep with an accumulator
C++
Rust

using Clock = std::chrono::steady_clock;     // monotonic; never wall-clock for dt
const double dt = 1.0 / 60.0;                  // fixed simulation step (seconds)
double accumulator = 0.0;
auto currentTime = Clock::now();

while (running) {
    auto newTime = Clock::now();
    double frameTime = std::chrono::duration<double>(newTime - currentTime).count();
    if (frameTime > 0.25) frameTime = 0.25;  // spiral-of-death clamp (Fiedler's example)
    currentTime = newTime;

    accumulator += frameTime;
    while (accumulator >= dt) {           // >=, not >: a full step is available
        previousState = currentState;     // keep the prior state for interpolation
        integrate(currentState, dt);       // every step is exactly dt
        accumulator -= dt;                // carry the remainder (< dt) to next frame
    }

    double alpha = accumulator / dt;       // 0..1: how far past the last step we are
    render(lerp(previousState, currentState, alpha));
}

Worked example

The simulation steps at 60 Hz (dt = 16.67 ms) and the display refreshes at 144 Hz, so each frame adds 6.94 ms.

    Frame 1: accumulator 6.94 ms, no step, alpha = 6.94 / 16.67 = 0.42.
    Frame 2: 13.89 ms, no step, alpha 0.83.
    Frame 3: 20.83 ms, one step, 4.17 ms left over, alpha 0.25.
    Frame 4 hitches for 40 ms: 44.17 ms, two steps, 10.83 ms left, alpha 0.65.

Every step was exactly 16.67 ms. Only the number of steps per frame changed.

Three lines carry the whole design, so be precise about each:

    while (accumulator >= dt) runs zero, one, or several steps this frame, depending on how much real time built up.
    accumulator -= dt leaves a remainder below dt in the accumulator. It is carried, not discarded. Throwing it away makes the simulation run slow.
    alpha = accumulator / dt is the fraction of a step the renderer sits past the last completed state, the blend factor for interpolation in §7.

Fixed timestep is not fixed frame rate

A fixed timestep means the simulation advances in constant dt increments. A fixed frame rate means the display refreshes at a constant rate. They are independent: the normal case is a fixed-timestep simulation running under a variable, display-driven frame rate. The 0.25 s clamp is Fiedler's example value, not a universal constant; pick yours from your worst tolerable hitch.

The widget runs the same bouncing object under three loops at once:
Render FPS 24
Inject a frame spike
Three tracks run the same ball from the same wall clock: orange is the variable-timestep loop, blue the fixed loop drawing its latest tick, and green the fixed loop with interpolation (its faint ghost is the latest tick). Drag the render-rate slider and press "Inject a frame spike". The simulation ticks at 16 Hz, deliberately coarse so the ticks are visible. At any render rate above that, the bare fixed loop judders: its drawn position only changes on tick boundaries, so some frames repeat a position and others jump, and its step count reads 0, 1 or 2. The interpolated loop runs the same ticks and moves as smoothly as the variable loop, because it draws the leftover accumulator as a blend. Below 16 FPS every track is choppy, since nothing can show motion between frames the display never draws. The spike adds 0.32 s to one frame. The variable loop takes all of it, plus the frame's own time, in one step; the fixed loops clamp the frame to 0.25 s and take it as four 1/16 s ticks (the blue readout shows "4 step(s) this frame"), so they jump a little less and fall slightly behind the wall clock.
Check Your Understanding Q1 of 2

After the inner loop runs, the accumulator holds a value less than dt. What do you do with it?
Leave it in the accumulator; it carries into next frame, and divided by dt it is the interpolation alpha.
Reset it to zero so each frame starts clean.
Add it as a final partial integration step.
0/2 correct

Wrong answers, and why: zeroing the accumulator loses time (slow sim); a final partial step is the semi-fixed variant that breaks determinism; and fixed timestep constrains the simulation, not the display or vsync.

[Back to The Game Loop & Time](index.md)
