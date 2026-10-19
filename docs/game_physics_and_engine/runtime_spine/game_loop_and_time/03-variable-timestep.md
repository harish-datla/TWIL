# Variable Timestep


The obvious fix for a player who jogs on one machine and sprints on another: measure how long the last frame took and scale every movement by that delta time, the real time since the previous frame, usually called dt and measured in seconds.
Variable timestep
C++
Rust

auto previous = Clock::now();
while (running) {
    auto now = Clock::now();
    double dt = std::chrono::duration<double>(now - previous).count();
    previous = now;
    update(dt);     // position += velocity * dt
    render();
}

Wall-clock speed is now consistent across frame rates, and for a lot of gameplay code this is fine. For the simulation core it brings three new failures:

    Non-determinism. Same inputs, different results. Different machines produce different dt sequences. A numerical integrator's error depends on the step size, so anything accelerating (gravity, drag, springs) lands somewhere different for a different sequence of steps, and floating-point rounding adds its own drift on top: a machine taking ten times as many steps accumulates more of it[2].
    Large-step instability. A frame spike hands update a huge dt, and an explicit integrator (code that advances position and velocity by one finite step using only the current state) can overshoot and blow up. This is integrator-dependent: explicit Euler is fragile, while semi-implicit Euler (update velocity first, then move with the new velocity) and higher-order schemes are more forgiving[4].
    Tunneling. A big step jumps a fast object clear past a thin wall between two discrete collision checks. A player running at 10 m/s moves 0.17 m per step at 60 Hz, less than a 0.2 m wall; a 0.25 s hitch moves them 2.5 m in one step, straight through it. That is a sampling problem, distinct from integrator instability.

Worked example

Drop a ball from rest for 1 second under gravity (9.81 m/s²) with semi-implicit Euler. The exact fall is 4.905 m. A player at 144 fps takes 144 steps and the ball falls 4.939 m; at 60 fps, 4.987 m; at 30 fps, 5.069 m. Same inputs, and the 144 fps and 30 fps machines disagree by 13 cm after one second.

Now a spring: stiffness 100 N/m on a 1 kg mass, stretched 1 m. Semi-implicit Euler stays bounded for steps under 0.2 s and is smooth at 1/60 s. One 0.25 s frame is past that limit: that single step flings the mass from +1 m to −5.25 m, five times its starting stretch. If the hitches keep coming, the next steps reach 21.3, −85.3 and 341 m.

[Back to The Game Loop & Time](index.md)
