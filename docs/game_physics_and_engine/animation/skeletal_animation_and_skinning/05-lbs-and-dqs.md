# LBS & DQS

Twist a character's wrist toward 180° and, with the standard skinning math, the forearm pinches to a point like the twisted end of a candy wrapper. Linear blend skinning (LBS, a.k.a. matrix palette skinning) is the equation above verbatim: blend the joint matrices by the weights, then transform. It's the most common method in real-time rendering[4] and the one glTF's skinning definition uses: a linear weighted sum of joint matrices[2]. But a weighted sum of matrices is not a rigid transform, so it has two artifacts:

    Candy-wrapper (twist): at a joint twisted toward 180°, the blend collapses volume. With one rotation the identity and the other a 180° twist, 0.5·I + 0.5·R is a rank-1 matrix that projects space onto the axis, pinching the cross-section to a point[6].
    Volume loss (bend): at a sharply bent elbow, linearly interpolating between two rotated positions cuts the corner, shrinking the inner volume.

Worked example

A forearm 5 cm in radius, twisted at the wrist, with a vertex halfway along weighted 50/50 between elbow and wrist. LBS scales that cross-section by cos(twist ÷ 2): at a 90° twist the radius drops to 3.5 cm, at 150° to 1.3 cm, and at 180° to zero. Dual-quaternion skinning keeps it at 5 cm throughout.
DQS fixes the twist, but it's not a free or universal upgrade

Dual-quaternion skinning (DQS, Kavan et al.) blends rigid transforms instead of matrices, so it preserves volume through twists with no candy-wrapper, at cost comparable to LBS[5]. But DQS has its own artifact, joint bulging on sharp bends (it over-preserves volume), it can't represent non-uniform scale or shear, and it needs antipodality handling (the q vs −q double cover from 3D Math). LBS stays the default; the common production fix for its artifacts is twist helper joints (extra forearm joints that each take part of the twist) and corrective blend shapes (sculpted fixes that fade in at certain joint angles)[4], not necessarily a switch to DQS.

The widget twists a forearm under both methods:
Twist
LBS
DQS
A forearm from the side with stripes painted on the skin, and its cross-section at the dashed line; the readout gives the mid-limb radius. Drag Twist toward 180°, then switch to DQS. The readout tracks cos(twist ÷ 2): 71% at 90°, 50% at 120°, 0% at 180°. Under LBS the cross-section stays round but shrinks toward a point on the twist axis: at the 50/50 weight the blended matrix scales the section by cos(twist/2), which is zero at 180°, where 0.5·I + 0.5·R is the rank-1 projection onto the axis. That pinch is the candy-wrapper. DQS keeps the full radius. The trade: DQS instead bulges at sharp bends (this demo isolates the twist, so the bend bulge isn't drawn), and the usual production fix for LBS is twist helper joints rather than DQS.
