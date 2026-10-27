# Capstone: A Complete 2D Game

The engine now has a game loop, sprite renderer, input, audio, and asset package. This capstone wires them into a small top-down arena shooter. A fixed-timestep simulation consumes input, pushes sounds through a lock-free ring, resolves collisions, and drives a state machine. The renderer batches an interpolated snapshot into a few draws.

Time: ~80 min  
Level: Senior  
Prerequisites: The 2D track—Game Loop, Input, Assets, 2D Renderer, and Audio.  
Stack: C++ & Rust

## What you’ll be able to do

- Start subsystems in dependency order and shut them down safely in reverse.
- Run a fixed-timestep simulation with interpolated rendering and a spiral-of-death guard.
- Choose a game-object model appropriate to the game’s scale.
- Keep simulation, renderer, and audio threads separated.
- Pass sound commands through a lock-free ring and drop instead of waiting when full.
- Reduce collision work with a uniform-grid broadphase.
- Drive menus and waves with a state machine and run the game headlessly.

## Sections

- [What Done Looks Like](01-what-done-looks-like.md)
- [The Shape of main()](02-the-shape-of-main.md)
- [Frame Anatomy](03-frame-anatomy.md)
- [The Game-Object Model](04-the-game-object-model.md)
- [World to Render](05-world-to-render.md)
- [Input and Audio](06-input-and-audio.md)
- [Physics in One Accumulator](07-physics-in-one-accumulator.md)
- [Assets and the State Machine](08-assets-and-the-state-machine.md)
- [The Whole Game, Headless](09-the-whole-game-headless.md)
- [What’s Next](10-whats-next.md)
