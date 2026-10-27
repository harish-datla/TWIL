# Pitfalls
Hard-cut transitions
Pop. A transition is a timed cross-fade (or inertialization).
Blending global matrices
Non-rigid in-betweens. Blend local TRS, then recompute globals.
2D blend as two 1D blends
Wrong off the axes. Use a real 2D scheme (gradient band).
Inertialization without velocity
Record only the position gap and the joint's speed snaps to the new clip's at the switch, a visible kink. Record the velocity difference too.
Layer without a mask
Upper and lower body fight. Mask which joints a layer drives.
Mismatched additive conventions
Build and apply the delta on the same side: reference⁻¹ · pose with base · delta, or pose · reference⁻¹ with delta · base. Mix them and the lean turns about the wrong axis.
Unscaled features
Metres swamp unit vectors in the distance. Divide each feature group by its standard deviation.
Matching the clip's last second
A frame without a full future can't be described. Leave it out of the search.
"Motion matching needs no authoring"
It needs a curated mocap database and tuned costs; game logic can stay a state machine.
FABRIK without the root reset
The chain drifts off its anchor. Reset the root in the backward pass.
Instant foot IK
Legs snap on every step change. Raycast + pelvis drop + leg IK, interpolated over time.
"FABRIK is exact"
It's iterative; it clamps on unreachable. Two-bone is the exact one.
