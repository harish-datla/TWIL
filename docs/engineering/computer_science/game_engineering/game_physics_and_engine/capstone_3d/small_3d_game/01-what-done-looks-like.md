# What "done" looks like

Picking the game is the first integration decision. A puzzle game would barely touch physics or AI, and a racing game's AI follows a racing line with no navmesh to path across. The capstone game is a third-person survival arena, a small design that forces every subsystem to fire without genre-specific tuning eating the budget:
The vertical slice

One level (a glTF scene with a baked navmesh), a player capsule (move + look, the character controller), waves of AI enemies that path and steer toward the player and attack in melee range, a dodge/attack combat loop, rendered with the deferred PBR pipeline + a shadow + TAA/bloom/tonemap, audio cues on hits and spawns, and an ImGui debug overlay showing the live frame profiler. Win = survive N waves; lose = health hits zero. It has the shape of a vertical slice (every system, end to end, on one small level) without the near-final polish a production slice carries: every subsystem fires once and nothing is tuned to ship.
