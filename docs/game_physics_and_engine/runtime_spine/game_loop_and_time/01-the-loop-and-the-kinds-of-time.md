# The Loop and the Kinds of Time


A game is a loop: read input, advance the simulation, draw a frame, repeat. The whole subject of this tutorial is the middle step, advancing the simulation, and specifically how much time each advance represents. Get that wrong and everything downstream (physics, animation, networking) inherits the error.

A pause menu has to stop the world while its own buttons keep animating; bullet time has to slow the world while the HUD runs at full speed. That takes more than one clock, so it helps to separate the kinds of time an engine juggles[1]. Real time is the wall clock. Simulation time is how far the game world has advanced, which can run slower than real time (a hitch), faster (fast-forward), or stop (pause) without the program freezing. Local time is per-system, like the playhead of one animation clip. This tutorial is about keeping simulation time correct and stable as real time does whatever the hardware makes it do.

Worked example

Bullet time runs the simulation at 25% speed. In 2 seconds of real time, simulation time advances 0.5 s, and a grenade's 3-second fuse needs 12 real seconds to go off. Open the pause menu for 10 real seconds and simulation time advances 0, while the menu's own fade-in animation plays its full local time.

[Back to The Game Loop & Time](index.md)
