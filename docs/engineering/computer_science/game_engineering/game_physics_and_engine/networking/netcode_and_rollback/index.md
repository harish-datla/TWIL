# Networking: Netcode & Rollback

Network latency, jitter, packet loss, bandwidth limits, and cheating make multiplayer state difficult to share. This tutorial covers UDP channels, client prediction, reconciliation, interpolation, lag compensation, and rollback.

Time: ~25 min  
Level: Senior  
Prerequisites: The Game Loop, Floating Point, and Compression.  
Phase: 12 · Networking

## What you’ll be able to do

- Choose UDP or TCP per channel and build acknowledgements and partial reliability over UDP.
- Choose lockstep, client-server, or rollback for a genre.
- Implement client prediction and replay-based reconciliation.
- Smooth remote players with an interpolation buffer.
- Rewind the world for lag compensation and size rollback resimulation windows.

## Sections

- [Why It’s Hard](01-why-its-hard.md)
- [UDP vs TCP](02-udp-vs-tcp.md)
- [The Three Models](03-the-three-models.md)
- [Lockstep & Determinism](04-lockstep-and-determinism.md)
- [Prediction & Reconciliation](05-prediction-and-reconciliation.md)
- [Interpolation](06-interpolation.md)
- [Snapshots & Delta](07-snapshots-and-delta.md)
- [Lag Compensation](08-lag-compensation.md)
- [Rollback](09-rollback.md)
- [Choosing a Model](10-choosing-a-model.md)
- [Pitfalls](11-pitfalls.md)
- [What’s Next](12-whats-next.md)
