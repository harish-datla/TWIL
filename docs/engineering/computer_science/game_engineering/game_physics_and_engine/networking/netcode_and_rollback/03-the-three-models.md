# The three models

An RTS with 1,500 units, a 64-player shooter and a two-player fighting game have very different problems: too much state to send, players who might cheat, and timing tight enough that a few frames decide a match. Three archetypes, each suited to a different genre. They differ in what crosses the wire (inputs vs state), who is authoritative (peers vs a server), and whether a peer waits for remote inputs or predicts them.
ModelWire / authorityGenre & tradeoff
Deterministic lockstepInputs only; peers (P2P)RTS. Tiny bandwidth, but needs perfect determinism + waits for the slowest peer
Client-server authoritativeState; server is truthFPS. Cheat-resistant; needs prediction to feel responsive
RollbackInputs only; peersFighting games. Lockstep that predicts remote inputs instead of waiting; needs determinism + cheap state save/restore

A fourth model sits between the first two: state synchronization runs the sim on both sides and sends inputs plus state, so the state corrects drift and perfect determinism isn't required[14].
