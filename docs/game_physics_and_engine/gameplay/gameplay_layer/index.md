# The Gameplay Layer

The gameplay layer turns engine capabilities into a game: modeling world objects, connecting their parts, ordering updates, managing lifetime, and moving definitions and behavior into data and scripts.

Time: ~80 min  
Level: Senior  
Prerequisites: The ECS tutorial and The Game Loop.  
Stack: C++ & Rust · Lua  
Phase: 10 · Gameplay

## What you’ll be able to do

- Choose between Unity-style, Unreal-style, and ECS composition.
- Connect components with direct references, observers, or event queues.
- Handle update-order dependencies with dependency ordering or double buffering.
- Spawn and destroy objects safely during a tick.
- Use generational handles for cross-frame references.
- Define enemy variants as data-driven prefabs and build them with a component factory.
- Embed Lua safely, including coroutines and hot reload.

## Sections

- [The Gameplay Layer](01-the-gameplay-layer.md)
- [Why Inheritance Failed](02-why-inheritance-failed.md)
- [Composition: Three Models](03-composition-three-models.md)
- [Component Communication](04-component-communication.md)
- [Update Order](05-update-order.md)
- [Lifetime & Handles](06-lifetime-and-handles.md)
- [Data-Driven Design](07-data-driven-design.md)
- [Embedded Scripting](08-embedded-scripting.md)
- [Pitfalls](09-pitfalls.md)
- [What’s Next](10-whats-next.md)
