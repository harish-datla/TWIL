# IK & root motion

The walk cycle was authored on flat ground, and now the character is on stairs: one foot floats above a step and the other sinks through it. Forward kinematics computes positions from joint angles (everything above). Inverse kinematics is the reverse: given a target (put the foot on the ground, the hand on the doorknob), find the joint angles that reach it. IK modifies the pose before the skinning palette is built.

    Two-bone (limb) IK is analytic: the interior elbow angle comes from the law of cosines, acos((l1² + l2² − D²) / (2·l1·l2)) for target distance D, then orient the triangle toward the target[9]. Clamp D to the reachable range first, or acos gets an argument outside [−1, 1]. The law of cosines fixes the elbow angle but not the plane: the triangle can spin freely about the shoulder-to-target line, so a pole (bend) vector picks which way the elbow points. Take a 0.3 m thigh, a 0.25 m shin and an ankle target 0.4 m from the hip. The argument is (0.09 + 0.0625 − 0.16) ÷ 0.15 = −0.05, so the knee's interior angle is 92.9°. A target 0.6 m away gives −1.38, outside acos's range; clamped to the 0.55 m reach, the leg straightens to 180°.
    General chains use iterative solvers: FABRIK places joints along lines, first forward from the end effector (set to the target) in to the base, then backward from the base (reset to its anchor) out to the tip, converging in few iterations with no rotation matrices or angles[10]; CCD rotates one joint at a time from the end inward.

Root motion vs in-place

In-place: the root stays at the origin and gameplay code moves the character. Root motion: the animation's root bone translates within the clip, and the engine drives the character's world transform from that root delta, so the visual stride matches the displacement[12]. A common cause of foot sliding is an in-place clip whose authored stride speed differs from the code-driven movement speed (a clip authored at 1.4 m/s moved at 1.6 m/s slides its planted foot 3.3 mm every 60 Hz frame); root motion removes that mismatch, and foot-locking IK handles other sources. Root motion has costs too: the animation, not gameplay code, now decides movement, and networked games have to replicate or predict it. Many games therefore keep locomotion in place and use root motion for discrete actions such as attacks and vaults.
Check Your UnderstandingQ1 of 2

You load a glTF character, animate it, and the mesh explodes into a tangled mess the instant playback starts. The most likely cause?
The inverse bind matrix is missing or applied on the wrong side; the skinning matrix must be jointGlobalAnimated times inverseBind.
The vertex weights need to be in the fragment shader instead of the vertex shader.
The skeleton has too many joints for a UBO.
0/2 correct

Wrong answers, and why: an exploding mesh on playback is the inverse-bind order/omission (not shader stage or UBO size); and mid-blend shrinkage is wrong rotation interpolation / blending globals (not the blend factor, and not a DQS issue).
