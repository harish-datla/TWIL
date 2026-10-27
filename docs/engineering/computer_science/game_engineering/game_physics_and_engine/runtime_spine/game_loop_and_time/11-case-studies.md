# Case Studies


If you ship on an existing engine, it already made these choices. Knowing which ones tells you what its physics settings do and why its docs warn about the things they warn about.
Unity
Fixed FixedUpdate plus rigidbody interpolation. Physics runs at Time.fixedDeltaTime, default 0.02 s (50 Hz)[7]. Rigidbody Interpolate blends the previous two physics updates and so renders one update behind; Extrapolate predicts ahead instead[8]. The spiral guard is Maximum Allowed Timestep[7].
Unreal
Variable tick with optional physics sub-stepping. The frame tick is variable. Sub-stepping is off by default; turned on in the project's Physics settings, it sub-divides each frame so no physics step exceeds Max Substep Delta Time, bounded by Max Substeps[9]. The sub-stepping is hidden from gameplay code.
Godot
Split processing callbacks. _physics_process runs at a fixed Physics FPS (default 60, so 16.667 ms) and _process runs at the variable frame rate[10]. Optional physics interpolation blends the current and previous transforms, explicitly rendering in the past[11].
Source
Fixed server tick. Source "simulates the game in discrete time steps called ticks," a default 15 ms step, so 66.67 ticks per second, decoupled from the client's frame rate[12].

[Back to The Game Loop & Time](index.md)
