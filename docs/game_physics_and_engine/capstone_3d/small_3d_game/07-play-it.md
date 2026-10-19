# Play it

A top-down canvas stand-in for the 3D arena. The loop is the real one: a fixed-timestep accumulator with interpolation. The other layers are small versions of the modules': A* on a grid with line-of-sight smoothing in place of the navmesh and funnel, arrive-style steering with separation, circle-versus-box collide-and-slide for the capsule, a pulse for each audio cue, and the wave/win/lose loop. Press start (or click the arena and press Enter), move with WASD / arrows, attack with space:
start / restart
render raw sim (no interpolation)
show AI layers
lag spike
Press start and play a wave, then try the toggles. Enemies run a reduced AI stack: the decision is fixed (chase, strike in melee range), then path, steer, resolve. The audio pulse in the top-right corner fires on each cue: an attack that lands, a wave that spawns. Render raw sim draws the latest sim state without interpolation; the stutter is clearest on a high-refresh display, where several frames repeat the same step. Lag spike adds 200 ms to one frame, so that frame runs about 13 steps instead of 1 (216.7 ms / 16.7 ms on a 60 Hz display): every enemy jumps, and the step counter's peak holds for a second. Show AI layers draws each enemy's remaining path corners (blue) and its steering vector (amber).
