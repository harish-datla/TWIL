# IK in depth

The animation was authored on flat ground, but the character is on a staircase, reaching for a ledge, or looking at the player. Forward kinematics gives positions from angles; inverse kinematics finds the angles to reach a target. Two-bone IK (covered in Skeletal Animation) is analytic and exact[14]: the law of cosines gives the knee or elbow angle from the two bone lengths and the distance to the target, and a pole vector picks which way the joint points. General chains use iterative solvers:

    FABRIK (Aristidou & Lasenby): a forward pass (end effector → root) and a backward pass (root → end effector), each placing joints on a line at the original bone lengths. In the authors' benchmarks it needed fewer iterations and less time than CCD and Jacobian methods[15].
    CCD (cyclic coordinate descent): rotate one joint at a time, end → root, to aim the end effector at the target; iterate. Simple and cheap per step, but each joint is solved in isolation, so it can stall in local minima and produce unnatural poses[16][15].
    Jacobian methods: linearize end-effector motion against joint angles and step (transpose, pseudo-inverse, or damped least squares); smooth, handles many DOF, but costlier per iteration and unstable near singular poses unless damped.

Iterative solvers are approximate, clamp on unreachable, and FABRIK must reset the root

Unlike two-bone, FABRIK and CCD converge over iterations (more iterations: closer, but more cost) and can't reach an unreachable target; clamp to max reach (FABRIK stretches the chain straight at the target in one pass). FABRIK's backward pass must reset the root to its anchor; skip it and the whole chain drifts off. Foot IK is a combination: raycast down per foot, lower the pelvis so the lower foot can reach its ground point, run two-bone leg IK to plant each foot, and aim the ankle to the surface normal[17]. Then interpolate the offsets over time; applied instantly, the legs snap every time the ground height changes. Look-at distributes rotation across spine joints with limits (no 180° head snaps). IK runs after the pose is produced and before the palette is built, adjusting that pose (usually with a blend weight) rather than generating one.
FABRIK (forward + backward passes; the backward pass resets the root)
C++
Rust

// positions[0] = root (anchored); positions[n-1] = end effector.
// boneLengths[i] is the distance from joint i to joint i+1, captured at bind time.
void solveFABRIK(std::vector<vec3>& positions, const std::vector<float>& boneLengths,
                 vec3 target, int maxIterations = 10, float tolerance = 1e-3f) {
    const size_t n = positions.size(); const vec3 root = positions[0];
    float reach = 0; for (float boneLength : boneLengths) reach += boneLength;
    if (length(target - root) > reach) {                 // UNREACHABLE: stretch straight (1 pass)
        for (size_t i = 0; i + 1 < n; ++i)
            positions[i+1] = positions[i] + normalize(target - positions[i]) * boneLengths[i];
        return;
    }
    for (int iteration = 0; iteration < maxIterations; ++iteration) {
        if (length(positions[n-1] - target) < tolerance) break;   // converged
        positions[n-1] = target;                        // FORWARD: end effector onto target, then inward
        for (int i = (int)n-2; i >= 0; --i)       // keep each joint on the line to its child, at bone length
            positions[i] = positions[i+1] + normalize(positions[i] - positions[i+1]) * boneLengths[i];
        positions[0] = root;                            // BACKWARD: reset the root (critical!), then outward
        for (size_t i = 0; i + 1 < n; ++i)
            positions[i+1] = positions[i] + normalize(positions[i+1] - positions[i]) * boneLengths[i];
    }
}

Worked example

Two 1 m bones lie along x, from a root at (0, 0), and the target is (0.5, 1.2), 1.3 m away and within reach. The forward pass puts the tip on the target, pulls the middle joint to (0.885, 0.277), and drags the root to (−0.07, −0.02), off its anchor. The backward pass puts the root back at (0, 0) and pushes outward: the middle joint lands at (0.954, 0.299) and the tip at (0.504, 1.192), 9.3 mm short. A second iteration brings the miss to 0.2 mm. Skip the root reset and the next forward pass starts from the dragged root, and the arm walks away from the shoulder.

The widget runs that loop on a five-bone chain:
step one iteration
animate
drag the target dot
A six-joint chain anchored at the white root, with its maximum reach drawn as a faint circle. The target starts outside that circle, so the chain loads stretched straight at it; drag the target inside, then out again. The chain runs FABRIK (forward then backward passes) to reach it, one iteration every 0.12 s, and the readout counts iterations for the current target and the end-to-target distance. A target well inside the circle typically settles within a pixel in a few iterations. Pull the target out of range and the chain stretches straight at it in a single pass, with the distance stuck above zero: the unreachable case, clamped to max reach.
