# The Naive Loop


The first loop everyone writes runs at the right speed on the machine it was written on. It advances the world by a fixed amount of work per iteration, with no notion of elapsed time:
The frame-rate-dependent loop
C++
Rust

while (running) {
    pollInput();
    update();        // advances by a fixed step of WORK, not time
    render();
}

The simulation speed equals the loop's iteration rate, which is set by the hardware and the current load[2]. A faster CPU runs the game faster; a slower one runs it in slow motion. It's why some early PC games became unplayably fast on later hardware.

Worked example

update() moves the player 0.1 m. At 60 iterations a second that's 6 m/s, a brisk jog. The same build on a machine that runs the loop 144 times a second moves them at 14.4 m/s, and on a laptop that manages 30, at 3 m/s.
Trap: vsync does not fix this

Capping the loop with vsync (waiting for the display's refresh before showing each frame) bounds the upper rate, so the coupling stops being visible on hardware fast enough to hit the cap. The cap is the monitor's refresh rate, though, so the same game runs 2.4× faster on a 144 Hz display than on a 60 Hz one (144 / 60), and a machine that can't keep up still under-runs and goes slow-motion. Vsync hides the symptom on some setups; it doesn't decouple simulation from frame rate.

[Back to The Game Loop & Time](index.md)
