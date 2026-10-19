# The 3D-Game Capstone

This capstone integrates the engine subsystems into a small playable 3D survival arena. It establishes the rules that keep the systems coherent: one fixed-step loop, explicit update order, a read-only renderer, and safe cross-thread handoffs.

Time: ~60 min  
Level: Senior · capstone  
Prerequisites: Most of the series.  
Stack: C++ & Rust  
Milestone: A small 3D game

## What you’ll be able to do

- Write the master frame loop with one accumulator and a fixed update order.
- Trace an enemy through decision, planning, steering, and resolution.
- Order deferred render passes and explain TAA, bloom, and tonemapping dependencies.
- Read a frame budget and recognize the spiral of death.
- Choose an appropriate networking model for the integrated simulation.

## Sections

- [What “Done” Looks Like](01-what-done-looks-like.md)
- [The Engine, Layer by Layer](02-the-engine-layer-by-layer.md)
- [The Master Frame Loop](03-the-master-frame-loop.md)
- [Update Order & the AI Stack](04-update-order-and-the-ai-stack.md)
- [Assembling the Render Frame](05-assembling-the-render-frame.md)
- [Making It a Game](06-making-it-a-game.md)
- [Play It](07-play-it.md)
- [The Frame Budget](08-the-frame-budget.md)
- [The Data Flow](09-the-data-flow.md)
- [If You Added Networking](10-if-you-added-networking.md)
- [What Each Module Gave](11-what-each-module-gave.md)
- [Retrospective & Pitfalls](12-retrospective-and-pitfalls.md)
- [What’s Next](13-whats-next.md)
