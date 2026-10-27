# Seeing the engine

An NPC walks into a wall, but only after the level has been running for ten minutes. Rebuilding with print statements and replaying ten minutes per guess is slow. Tooling replaces guessing about the running game with looking at it. Game Engine Architecture gives these tools a chapter of their own: logging and tracing, debug drawing, in-game menus, an in-game console and in-game profiling[1]. Before any GUI, the cheapest tools give the most value for the effort:

    Logging / tracing: timestamped, categorized, filterable; the first thing you reach for.
    Asserts: an invariant written as executable code; a failed assert stops the program where the broken assumption is first checked, usually much closer to the bug than the crash it would have caused.
    Debug draw: "printf for geometry". Draw the navmesh, the AI paths, the collision shapes and the light bounds right in the world. For the NPC above, drawing its planned path shows at a glance whether the path goes through the wall or the steering ignores it.

The rest of this module is the next tier: an in-engine UI to tweak live state, profilers (tools that measure where the time goes) to measure, and frame capture to dissect the GPU.
