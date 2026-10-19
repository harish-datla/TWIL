# Assets and the State Machine

Mount the cooked package at startup and load the atlas, sounds, and level. Load-all-up-front is appropriate for this arena; open-world games need streaming and eviction.

Use explicit states such as `Menu`, `Playing`, `Paused`, and `GameOver`. Menu transitions to playing on Enter; playing pauses on Escape, resumes from pause, and enters game over when the ship dies; game over returns to menu on Enter. Paused continues to render the last snapshot but performs zero simulation steps.

Keep dataflow one-way: input feeds simulation, simulation produces render descriptions and audio commands, the renderer consumes descriptions, and the audio thread consumes commands. Never let simulation call the GPU directly; that coupling prevents headless tests and clean interpolation.
