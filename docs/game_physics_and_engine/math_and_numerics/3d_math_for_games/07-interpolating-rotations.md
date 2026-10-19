# Interpolating Rotations


A character blends from a run into a jump over a fifth of a second, and every bone's rotation has to pass smoothly from one orientation to the other. Blending two orientations is the daily job of an animation system. There are three contenders, and the "obvious correct" one is not always the right pick.

SLERP (spherical linear interpolation, Shoemake 1985[11]) walks the great-circle arc (the shortest path across the sphere's surface) between two unit quaternions at constant angular velocity, a steady turn rate from start to end:
eq. 6 · SLERP slerp(q₀, q₁, t) = sin((1−t)Ω) q₀ + sin(tΩ) q₁sin Ω , cos Ω = q₀ · q₁

In words: Ω (omega) is the angle between the two orientations, read straight off their dot product (cos Ω = q₀ · q₁). The two sine weights slide from all-q₀ at t = 0 to all-q₁ at t = 1, and because they ride the curve of the sphere rather than a straight chord, the blend turns at a steady rate the whole way across.

LERP interpolates the components in a straight line: a chord through the sphere, not along it. The result is not unit length and its rotation speed is non-uniform. NLERP is LERP followed by a renormalize; it rides the same arc as SLERP but not at constant angular velocity, running faster in the middle and slower at the ends[12].
When the difference actually matters

For small angular steps (adjacent animation keyframes, per-frame blends), NLERP is indistinguishable from SLERP, cheaper, and commutative, which is why many animation runtimes use it[12][13]. SLERP's constant-velocity property only becomes visible across large single interpolations, like a slow camera orbit over a wide arc. The framing (Blow, after Shoemake) is a three-way tradeoff: you want commutativity, constant velocity, and minimal torque, but you can't have all three. SLERP gives the last two; NLERP gives commutativity and minimal torque[13].

Worked example

Blend from no rotation, q₀ = (1, 0, 0, 0), to a 90° turn about y, q₁ = (0.707, 0, 0.707, 0). Then cos Ω = q₀·q₁ = 0.707, so Ω = 45°, half the turn. At t = 0.5 both sine weights are sin 22.5° / sin 45° = 0.541, giving (0.924, 0, 0.383, 0): a 45° turn, exactly halfway. LERP's midpoint is 0.5(q₀ + q₁), which has length 0.924, not 1.

At t = 0.25 SLERP gives 22.5° and NLERP 21.6°, 0.9° behind. For a 150° turn the same t gives 37.5° against 33.0°, 4.5° behind. The gap grows with the angle, which is why NLERP is safe between adjacent keyframes and visible on a wide camera swing.

The widget animates t from 0 to 1 over real seconds and draws all three paths plus a live angular-velocity strip. The circle is a slice through quaternion space, where a rotation of θ puts q₁ at θ/2 from q₀ (the half angle of eq. 5). Crank the angle to make the methods diverge, and toggle off the shortest-path fix to see the classic bug:
Angle 150°
take the shortest path
Play
Pause
The circle is a slice of quaternion space: q₀ on the right, q₁ at half the rotation angle, and three dots racing along the three paths, with each one's turn rate plotted in the strip below. At the starting 150° the readout shows q₀·q₁ = 0.26; drag the angle to 240° and it reads −0.50, and with the shortest-path box ticked the rotation traveled is 120°, not 240°. LERP's tip cuts inside the circle (non-unit). The velocity strip compares SLERP (flat) against NLERP, whose curve bulges in the middle, and LERP shares that exact profile, since renormalizing changes the radius, not the angle. Only SLERP holds a constant angular velocity. At small angles the three are nearly identical; the gap only opens up as you widen the angle. Past a 180° rotation, q₀·q₁ < 0: with the fix on, the blend heads for −q₁ (hollow marker) and turns 360° − θ the short way; turn the fix off and it turns the full θ the long way around (the readout shows the rotation traveled).
Check Your Understanding Q1 of 2

Your camera SLERP occasionally whips the long way around to the same orientation. The fix?
If q₀ · q₁ < 0, negate one quaternion before interpolating.
Normalize both quaternions before calling SLERP.
Switch from SLERP to LERP.
0/2 correct

Wrong answers, and why: normalizing the inputs doesn't address which arc you travel (the double cover does); dropping to LERP trades the long-way bug for a non-unit, variable-speed result. And NLERP is the one without constant angular velocity, so that can't be its advantage.

[Back to 3D Math for Games](index.md)
