# Choosing a model

You're starting a multiplayer game and have to commit to a model before writing the simulation, because lockstep and rollback both demand determinism from day one. Each model fits a genre's constraints:

    Lockstep → RTS: huge unit counts, P2P, bandwidth-bound, can pay the determinism price.
    Client-server + prediction/interpolation/lag-comp → FPS: needs an authoritative, cheat-resistant server.
    Rollback → fighting games: 2-player P2P, small deterministic state, latency-hiding above all.

The engine seam

One common arrangement: a network thread receives and parses packets and hands inputs/snapshots to the simulation thread over a lock-free SPSC queue, so wire I/O never stalls the fixed-timestep loop; plenty of shipped games instead poll non-blocking sockets at the top of each frame. Either way, every model here sits on top of that fixed tick and the determinism discipline from the Floating Point tutorial.
Check Your UnderstandingQ1 of 2

With client-side prediction on, your character feels instant, but a teammate insists you must snap straight to the server position every update or you will desync. What is the accurate correction?
On a server update you snap to the authoritative state, then REPLAY your unacknowledged inputs on top; pure snapping would throw away the prediction and stutter every packet.
You should not predict your own character at all; let the server move it.
Prediction and interpolation are the same system; use one for everything.
0/2 correct
