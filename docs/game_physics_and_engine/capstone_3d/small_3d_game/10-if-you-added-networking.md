# If you added networking

A friend asks for co-op. The good news is that the sim is already a self-contained simulate(world, input, dt) on a fixed timestep, the structural precondition netcode needs. Lockstep would exchange input per tick and run the identical simulate on every peer; rollback would snapshot world every tick, predict the remote player's input, and on a misprediction restore the snapshot and re-simulate to the present with the corrected inputs.

Worked example

Say the remote player's input arrives 4 steps late. A rollback restores the snapshot from 4 steps ago and re-simulates those 4 steps plus the current one inside a single frame. At the frame-budget model's default of 5.7 ms per sim step, that's 5 × 5.7 = 28.5 ms, well past the 16.7 ms budget before rendering even starts. A rollback game needs a step several times cheaper than this one.
The blocker is determinism, not architecture

A fixed step is necessary but not sufficient: float results can differ across compilers and CPUs, so making this net-ready is a determinism project (fixed-point or tightly controlled float), not a drop-in. This capstone is single-player, and its float nondeterminism is fine: bit-determinism starts to matter with lockstep, rollback, or replays recorded as inputs. Don't claim it's net-ready; claim its shape is.
