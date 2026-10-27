# Quaternions


An animation system stores thousands of bone orientations and blends between them every frame, and a camera has to turn smoothly through any pose, including straight down. Both need an orientation format that composes cheaply and has no dead spots. A quaternion is four numbers that encode an orientation. A unit quaternion q = (w, x, y, z) represents a rotation of angle θ about a unit axis n as:
eq. 5 · axis-angle to quaternion q = ( cos(θ/2), n sin(θ/2) )

In words: a quaternion's four numbers are not arbitrary: pack the cosine of half the angle into the first slot, and the axis scaled by the sine of half the angle into the other three. The orientation is stored directly, with no sequence of separate spins to go wrong, which is exactly what dodges gimbal lock (next subsection).

The half-angle is intrinsic, not a quirk: it falls out of the sandwich product used to rotate a vector, v′ = q v q⁻¹ (with v written as a pure quaternion, one whose real part is 0, and q⁻¹ = the conjugate for unit q: keep w, negate x, y and z).

Worked example

A 90° turn about +y: θ/2 = 45°, so q = (cos 45°, 0, sin 45°, 0) = (0.707, 0, 0.707, 0), and its conjugate is (0.707, 0, −0.707, 0).

The sandwich q (0, 1, 0, 0) q⁻¹ rotates the x axis to (0, 0, −1), which is where a right-handed 90° turn about y sends it. Negate all four numbers, (−0.707, 0, −0.707, 0), and the sandwich gives the same (0, 0, −1): the double cover below.

Two properties matter constantly. First, double cover: q and −q are the same rotation. That's harmless when you apply a quaternion and decisive when you interpolate two (next section). Second, the budget: four floats versus nine for a 3×3 matrix, and composition is one quaternion product versus a matrix multiply.
Versus Euler angles, and gimbal lock

A camera rig built from yaw, pitch and roll works until the player looks straight down; then yaw and roll start doing the same thing. Euler angles store an orientation as three sequential rotations (yaw, pitch, roll). They read nicely in an inspector and fail in a specific way:
What gimbal lock actually is

Gimbal lock is the loss of one degree of freedom that happens when the middle rotation of a yaw-pitch-roll sequence reaches ±90° (for sequences that reuse an axis, such as Z-X-Z, the lock is at 0° and 180°), which lines up the first and third rotation axes so they spin about the same direction[10]. It is a property of the three-angle representation, not of 3D rotation. Quaternions and rotation matrices don't gimbal-lock because they don't decompose orientation into three sequential axis rotations.

Gimbal lock is not "caused by quaternions being missing" or "by 90° rotations in general"; it is the middle-axis degeneracy of a three-angle sequence. Quaternions trade that singularity for an ambiguity: q and −q are the same rotation, which matters when interpolating (§7). And a quaternion built from Euler angles inherits their lock at that pose; the benefit comes from composing rotations as quaternions instead of rebuilding them from three angles.

Worked example

Build orientations as R = Ry(yaw) · Rx(pitch) · Rz(roll), the order the widget below uses. At pitch 90°, yaw 30° with roll 0° and yaw 0° with roll −30° give the same matrix: both sliders now turn the model about one line, and one direction of rotation is gone.

At pitch 80° the same two settings give different matrices. Nothing about 30° is special; the middle angle reaching 90° is.

The rig below is three nested gimbals driven by Euler angles. Drive the middle ring to 90° and the outer and inner rings snap parallel; yaw and roll then produce the same motion. Flip to the quaternion track and each slider instead turns the model about its own axis, composed as a quaternion product, so the three sliders move it in three independent directions at any pose:
Yaw 0°
Pitch 0°
Roll 0°
Euler gimbals
Quaternion
The three rings are the yaw, pitch and roll gimbals around the model, and the text at the top says how many independent axes are left. Push pitch to 90°, then move yaw and roll; then switch to the quaternion track and do the same. At pitch = ±90° the yaw and roll rings are coplanar: a readout flags "1 DOF lost," and dragging yaw or roll now rotates the model the same way. The cause is the middle axis hitting 90°, not the magnitude of any single rotation. The quaternion track starts from the current pose and applies each slider's change as a turn about the model's own axis (q ← q · Δq); those axes stay perpendicular, so pitch the model 90° there and yaw and roll still differ.
Versus rotation matrices, and drift

Both represent the same rotations, and the GPU ultimately wants a matrix for the vertex transform, so quaternions are usually converted to a matrix at the end of the chain. Pick by operation: quaternions are cheaper to store, compose, interpolate, and renormalize; matrices are cheaper to apply to many vectors, which is why linear-blend skinning shaders usually consume matrices.

Repeated quaternion multiplication accumulates floating-point error, so a quaternion that should stay unit-length slowly drifts off the unit sphere; left uncorrected, it introduces scale/shear distortion when converted to a matrix. The fix is to renormalize periodically, every few multiplications, not necessarily after every single one. Rotation matrices drift the same way (they lose orthonormality) and need re-orthonormalization rather than mere column-normalization.

Worked example

Compose a 0.5° turn about a fixed axis onto a float quaternion once per frame and never renormalize. In one float32 run its length was off by about 5 × 10−6 after 1,000 frames and 4 × 10−4 after 100,000, about 28 minutes at 60 fps.

The error grows steadily rather than averaging out. A renormalize every few frames (compute the length, divide all four numbers by it) keeps it at rounding level.

[Back to 3D Math for Games](index.md)
