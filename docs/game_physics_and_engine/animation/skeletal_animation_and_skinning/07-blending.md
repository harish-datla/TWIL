# Blending

The character walks at 1.4 m/s and runs at 4 m/s, and the player has the stick halfway. Neither clip is right; a mix of the two is. Blending combines poses per joint, in local space, then recomputes globals[1]. A cross-fade interpolates each joint's local rotation by slerp and translation by lerp over the fade. Blend trees parametrically mix clips by 1D (speed) or 2D (speed × direction) parameters; Unreal calls them Blend Spaces and Unity calls them Blend Trees. Additive blending applies a pose difference (a clip authored relative to a reference) on top of a base, for aim offsets and hit reactions layered over locomotion.

Worked example

A blend weight of 0.3 toward run, with the thigh's local rotation at 20° in the walk and 40° in the run about the same axis, gives 20° + 0.3 × 20° = 26°. An additive aim offset authored as +15° from its reference pose then composes on top, to 41°, whether the base pose came from the walk, the run or a crouch.
Blend in local space, and additive is a difference

Don't blend global joint matrices: that gives non-rigid, broken in-betweens (the matrix-averaging problem again). Blend local TRS per joint (slerp the rotations), then walk the hierarchy. Additive is base + delta per joint (compose the delta rotation, add the delta translation), not a second clip played on top; that's why one aim-offset clip composes over walk, run, and crouch without authoring every combination.

The widget blends one joint between two poses with three interpolation methods:
Blend
slerp (correct)
lerp euler
lerp matrix
A limb blending from pose A (blue) to pose B (yellow) and back on its own; drag Blend to hold it, and switch between the three methods. The poses sit at 150° and −150°, 60° apart across the ±180° seam. Slerp takes the short arc at full length. Lerp euler interpolates the raw angle values, which are 300° apart, so the limb swings the long way through 0°; in 3D, per-axis angle lerp also leaves the rotation's great-circle path and misbehaves near gimbal lock. Lerp matrix shrinks the limb at the midpoint, to cos 30° = 87% of its length, because the interpolated matrix isn't rigid, the same failure as LBS. Quaternion slerp or nlerp, with the sign check, is the standard cheap fix.
