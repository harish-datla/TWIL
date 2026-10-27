# The master frame loop

On a 144 Hz monitor the game renders 144 frames a second, on a slow laptop maybe 40, and the simulation has to behave the same on both. The loop is the single integration point: the whole engine hangs off the fixed-timestep loop from The Game Loop. One accumulator drives the fixed sim, the render interpolates with alpha[2] and only reads the sim, and the sim never calls the GPU.

Worked example

With dt = 1/60 s = 16.7 ms, a frame that took 25 ms arrives with 5 ms already left in the accumulator: 30 ms in all. One step runs and leaves 13.3 ms, less than a step, so the loop stops and renders with alpha = 13.3 / 16.7 = 0.8, drawing every object 80% of the way from its previous position to its current one. A one-second stall (a breakpoint, a window drag) is clamped to 0.25 s, so the next frame runs 0.25 × 60 = 15 catch-up steps instead of 60.
The master loop (the spine that ties every subsystem together)
C++
Rust

const double dt = 1.0 / 60.0;        // the fixed sim step
double accumulator = 0.0;

while (running) {
    platform.pumpEvents(input);                  // OS events -> this frame's input snapshot (Input module)
    double frameTime = clock.tick();             // seconds since the last frame, from a monotonic clock
    accumulator += std::min(frameTime, 0.25);    // clamp = the spiral-of-death guard (Fiedler)

    while (accumulator >= dt) {                  // 0, 1, or N fixed steps, ONE accumulator
        world.previousTransforms = world.transforms;   // keep last step's transforms for interpolation
        simulate(world, input, dt);              // the whole fixed step (order below)
        accumulator -= dt;
    }
    double alpha = accumulator / dt;             // 0..1 blend between previous and current
    renderFrame(world, alpha);                   // READ-ONLY: deferred PBR + shadows + skinned meshes + post
    imguiOverlay(profiler);                      // debug UI on top of the final image (Tooling module)
    present();
}

void simulate(World& world, const Input& input, double dt) {
    applyInput(world, input);          // 1. resolve player actions
    aiTick(world, dt);                 // 2. behavior trees -> navmesh -> steering: desired velocities
    gameplayUpdate(world, dt);         // 3. component logic, events, spawns
    physicsStep(world, dt);            // 4. solver + character controllers resolve the motion
    animationUpdate(world, dt);        // 5. advance clips, sample poses from the resolved motion
    reapDestroyed(world);             // 6. deferred destruction at step end
}

Three integration rules

One accumulator, not N. Physics, AI, and animation are all things the one simulate(dt) does. Nest a second accumulator inside it (the physics tutorial runs one in isolation), feed it the frame time, and physics double-steps. A subsystem that needs a different rate derives it from the one step: AI every third step, physics as a fixed number of substeps. Render is read-only. No advancing an animation timer or moving an entity during draw; render consumes lerp(previous, current, alpha) for positions and slerp (or nlerp) for rotations. A raw component-wise lerp of a rotation gives a non-rigid, shrunken in-between, the same failure as LBS in Skeletal Animation. The sim never calls the GPU. It produces a scene snapshot the renderer consumes, which buys headless tests, clean interpolation, and a sim with the shape netcode needs.

Worked example

Nest a second accumulator for physics inside simulate and feed it the frame time. A 33.3 ms frame runs simulate twice, and each call adds 33.3 ms to the physics accumulator, so physics takes 4 steps of 16.7 ms while the game advanced 2: physics runs at double speed. And lerping a rotation component-wise goes wrong too. Halfway through a 90° turn about the vertical axis, the lerp of the two 2×2 rotation blocks is [[0.5, −0.5], [0.5, 0.5]]: it points the right way, 45°, but each column has length √(0.5² + 0.5²) = 0.71, so the model is drawn at 71% of its size for that frame. Slerp gives the same 45° at full size.
