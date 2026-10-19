# Decoupling Sim from Render


A player with a 144 Hz monitor should see 144 frames a second, and a player on a 30 fps laptop should still get the same simulation. The accumulator already decoupled the two rates, so neither one dictates the other. The inner loop runs the simulation zero, one, or several times per rendered frame; rendering happens once at the end, as often as the display allows. Unity says the same about its fixed step: FixedUpdate is called "zero, one, or multiple times per frame depending on the frame rate"[6], and Godot splits a fixed-rate _physics_process from a variable-rate _process[10].

Worked example

Unity's default fixed step is 0.02 s, 50 Hz. At 144 fps that's 50 / 144 = 0.35 fixed steps per rendered frame: FixedUpdate runs on about one frame in three and is skipped on the rest. At 30 fps it's 50 / 30 = 1.67 per frame, so it runs once on some frames and twice on others.
Where input fits, and why it is a determinism risk

In the single-threaded loop above, input is sampled once per rendered frame and fed to whatever simulation steps run that frame, not once per simulation tick (one fixed step). That is fine for most games and is itself a small source of input-timing variation. Lockstep and rollback netcode (where every machine runs the same simulation from exchanged inputs, rewinding when a late input arrives in rollback's case) tie input to ticks instead, which the networking modules cover.

[Back to The Game Loop & Time](index.md)
