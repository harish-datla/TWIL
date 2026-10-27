# What Done Looks Like

The capstone is the connective tissue between subsystem tutorials: a `main()` that starts them, a frame that coordinates them, and a small game that exercises all of them. At the architecture level, the Game layer may call down into every lower layer; nothing below should call back into the Game layer.

The example is a top-down arena shooter: steer a ship, spawn enemies in waves, fire bullets, resolve collisions, play event sounds, and track a score. It deliberately uses symmetric circle/AABB-style collision rather than platformer-specific edge cases.

A busy moment can contain one ship, 20 enemies, and 30 bullets: 51 sprites. With one atlas, this is one draw of 51 quads. The fixed simulation runs 60 times per second regardless of display refresh, while rendering happens once per refresh.
