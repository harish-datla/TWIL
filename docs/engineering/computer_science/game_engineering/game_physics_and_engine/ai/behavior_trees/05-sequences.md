# Sequences

A guard should fire only if it has ammo and has finished aiming. If any step fails, it shouldn't carry on to the next one. That "do these in order, stop at the first failure" pattern is a sequence.

A Sequence is AND logic. It executes its children left to right. If a child returns SUCCESS, the sequence moves to the next child. If a child returns FAILURE, the sequence stops and returns FAILURE. If all children return SUCCESS, the sequence returns SUCCESS. If any child returns RUNNING, the sequence returns RUNNING.

The mental model: "do A, then B, then C. If any step fails, abort." A patrol sequence might be: [HasWaypoint?, MoveToWaypoint, Wait(2s), NextWaypoint]. If HasWaypoint? fails, the entire patrol sequence fails and the parent selector tries something else.

Worked example

The patrol sequence on a frame where the guard is mid-walk: HasWaypoint? returns SUCCESS, so the sequence moves on; MoveToWaypoint returns RUNNING, so the sequence stops there and returns RUNNING. Wait(2s) and NextWaypoint aren't ticked this frame. Two frames later the walk ends: MoveToWaypoint returns SUCCESS and the sequence goes straight on to Wait(2s), which returns RUNNING for the next two seconds.
