# Pitfalls
"Never use TCP"
Never for time-critical data. TCP is fine for login/chat/downloads.
Lockstep sending state
It sends inputs; that's the bandwidth win. State is the snapshot model.
"Fixed timestep gives determinism"
Necessary, not sufficient. Floats diverge across machines.
Reconciliation that only snaps
Replay the unacked inputs, or you stutter every packet.
Predicting other players
In the snapshot model, interpolate them (render the past); rollback (§9) is the model that predicts them.
Delta vs the latest sent snapshot
Delta against the last ACKED snapshot, not the latest sent.
"Rollback is lag-free"
It hides latency; mispredictions pop. Needs determinism + cheap save/load.
One model for everything
Lockstep/RTS, client-server/FPS, rollback/fighting. Pick per constraints.
