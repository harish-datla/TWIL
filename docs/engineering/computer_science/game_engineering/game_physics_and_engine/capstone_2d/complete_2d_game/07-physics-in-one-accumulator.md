# Physics in One Accumulator

Physics is one operation inside `simulate(dt)`. Do not nest a second accumulator inside the physics module: handing the outer frame time to an inner accumulator counts elapsed time repeatedly and makes physics run ahead during catch-up. The game loop owns the timestep and passes `dt`.

For a bullet of radius 2 at `(10,10)` and enemy radius 8 at `(18,14)`, compare squared distance `8² + 4² = 80` with `(2+8)² = 100`; they collide without a square root.

## Uniform-grid broadphase

Bucket enemies into square cells, then test each bullet only against its own cell and eight neighbors. Cell size must be at least the largest contact distance; with enemy radius 10 and bullet radius 3, 13 px is the minimum for a 3×3 search. A practical 64 px grid over an 800×600 arena is 13×10 cells. A bullet at `(300,200)` is in cell `(4,3)` and searches nine cells instead of every enemy.

Too-small cells miss contacts unless the search widens; too-large cells save little over brute force. Crowding also reduces the benefit. Sweep-and-prune or AABB trees can adapt better to clumps and mixed sizes.
