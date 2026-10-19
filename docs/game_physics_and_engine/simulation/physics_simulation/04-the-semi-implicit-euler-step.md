# The semi-implicit Euler step

A crate dropped from rest should start falling on the first frame. The complete integration step for a single rigid body, linear part first, is:
SEMI-IMPLICIT EULER vnew = v + Fm · dt
xnew = x + vnew · dt

In words: first change the velocity by the acceleration over one step, then move the body by that new velocity over one step.

Worked example

A crate starts at rest under gravity, 9.81 m/s², with dt = 1/60 s. The first step sets its velocity to 9.81 ÷ 60 = 0.1635 m/s downward and then moves it 0.1635 ÷ 60 = 2.7 mm. Explicit Euler would move it by the old velocity, zero, and the crate would hang in place for one frame.

Position is updated with the new velocity. Rotation follows the same pattern: angular velocity omega is updated by the torque transformed by the inverse inertia tensor, and the orientation quaternion is advanced by omega. The code below drops the gyroscopic term ω × (Iω), as game engines commonly do by default; PhysX, for example, only adds it when a flag asks for it[12].
semi-implicit Euler step
C++
Rust
Copy

// Semi-implicit Euler integration for a rigid body.
// Velocity is updated FIRST, then position uses the new velocity.
void integrate(RigidBody& body, float dt) {
    if (body.inverseMass == 0.0f) return;  // static body, skip

    // --- Linear ---
    // accumulate gravity and any applied forces into acceleration
    Vec3 linearAccel = body.force * body.inverseMass;
    body.linearVelocity += linearAccel * dt;

    // position advances using the NEW velocity (symplectic)
    body.position += body.linearVelocity * dt;

    // --- Angular ---
    // world-space inverse inertia times torque = angular acceleration.
    // The gyroscopic term omega x (I * omega) is left out, as many game
    // engines do by default; spinning bodies lose a little realism.
    Vec3 angularAccel = body.inverseInertiaTensorWorld * body.torque;
    body.angularVelocity += angularAccel * dt;

    // integrate orientation quaternion
    // dq/dt = 0.5 * omega * q  (quaternion multiplication)
    // spin is the pure quaternion (w = 0, xyz = omega)
    Quaternion spin(0.0f,
        body.angularVelocity.x,
        body.angularVelocity.y,
        body.angularVelocity.z);
    body.orientation += (spin * body.orientation) * (0.5f * dt);
    body.orientation.normalize();  // the additive update drifts off unit length

    // clear per-frame accumulators
    body.force  = Vec3(0.0f);
    body.torque = Vec3(0.0f);
}

