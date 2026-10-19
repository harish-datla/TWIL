# The engine, layer by layer

The renderer needs a window before it can make a swapchain, and the asset loader needs the job system before it can read a file in the background. Start them in the wrong order and the first frame dereferences something that doesn't exist yet. The series built Gregory's runtime architecture bottom-up[1]; the capstone stacks it back together. Subsystems start up in dependency order and shut down in reverse, so the renderer releases its GPU resources before the allocators and the window it depends on go away:

    Platform & core: the window, the memory model + allocators, the job system, lock-free queues.
    Resources: asset pipeline, streaming, compression.
    The frame spine: the loop, input.
    Runtime systems: the renderer, animation, physics, audio.
    Gameplay & AI: the object model, behavior trees, navmesh + steering.
    The game on top, calling down, called by nothing.

This is one common decomposition (layered, deferred, fixed-step). Forward+, an entity-component-system everywhere, and a variable tick with fixed physics substeps are all valid alternatives.
