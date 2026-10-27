# Combining it all

Each stage above is easy to get right on its own; the bugs appear when they run in the wrong order, such as IK before layering, which lets the reload layer overwrite the planted arm. The full per-frame animation pipeline, end to end:

    A state machine or motion matching produces a base local pose.
    Layers blend on top (masked override + additive deltas).
    IK adjusts the result (foot plant, look-at, hand reach), modifying the pose.
    Recompute globals from the local pose (walk the hierarchy).
    Build the matrix palette and skin in the vertex shader (the Skeletal Animation module).

Every stage here is upstream of the palette; the skeletal module is where it all lands. The behavior tree (or other AI) sits one level up again: it sets the parameters and conditions that drive the state machine's transitions.
Check Your UnderstandingQ1 of 4

Switching from walk to run, your character snaps to the run pose with a visible jolt. What is wrong, and the rule?
The transition is a hard cut; it should be a timed cross-fade between the two states blended per joint in local space over a blend duration (or use inertialization).
The two clips have different lengths; resample them to the same length.
Blend the global matrices of the two states and the pop goes away.
0/4 correct

Wrong answers, and why: a transition pop is a missing cross-fade (not clip length; and you blend local TRS, never global matrices); FABRIK is iterative/approximate with a root-reset in the backward pass (it isn't exact, and you clamp unreachable targets rather than stretch bones); inertialization samples only the destination; and feature scaling balances the groups rather than speeding up the search.
