# Path to motion: steering

The guard now has a list of corners. Snap its velocity straight at each one and it turns instantly, runs at full speed into the last corner and stops dead, which looks robotic. Pathfinding is global and discrete (which polygons, computed occasionally). Steering is local and continuous (how to accelerate this frame, every frame). The path supplies the corners (from the funnel); steering produces smooth acceleration toward the current one using Reynolds' simple vehicle model[4]: a steering force clamped to a max, integrated into a velocity clamped to a max speed. In an engine, steering stops at that velocity: the character controller (§8) moves the agent and owns its position. The widgets below have no physics, so they advance position from the velocity themselves.
Don't conflate pathfinding with steering, and arrive is not seek

Pathfinding is global, discrete, and infrequent; steering is local, continuous, and per-frame. Steering executes on top of a path; it doesn't replace it. And the behaviors differ precisely: seek wants full speed at the target, so it arrives fast, overshoots, and turns back for another pass (Reynolds compares it to a moth buzzing a light bulb); arrive brakes by ramping desired speed down to zero inside a slowing radius. Wander is a coherent random walk on a circle projected ahead (not per-frame noise); pursue seeks the target's predicted position, not its current one.

Worked example

The widget below uses a max speed of 150 px/s and a slowing radius of 120 px. Arrive's desired speed is 150 × distance / 120, capped at 150: full speed beyond 120 px, 75 px/s at 60 px, 15 px/s at 12 px, and 0 on the target. Seek's desired speed is 150 px/s at every distance, so it reaches the target still moving at full speed and has to turn around.

The target hops to a new spot every few seconds (or drag it yourself). Compare seek, which overshoots, with arrive, which brakes, and with wander:
seek
arrive
wander
Switch between seek, arrive and wander. In seek and arrive the target hops every 3.5 s unless you've dragged it in the last 6 s. Seek reaches the target at full speed (it always wants full speed at the point), overshoots, and loops back; since the steering force is proportional to the velocity error, the loops shrink until the agent jitters on the point instead of stopping. Arrive ramps speed down inside the slowing radius and stops cleanly. Wander meanders smoothly. Drag the target mid-flight and the steering re-aims.
Steering behaviors: seek (overshoots) vs arrive (brakes)
C++
Rust

// clampMag truncates a vector to a max length. Reynolds' simple vehicle (mass = 1).
Vec2 seek(const Agent& a, Vec2 target) {
    Vec2 desired = normalize(target - a.position) * a.maxSpeed;  // full speed at the target
    return clampMag(desired - a.velocity, a.maxForce);          // no braking -> overshoots
}
Vec2 arrive(const Agent& a, Vec2 target, float slowingRadius) {
    Vec2 offset = target - a.position; float dist = length(offset);
    if (dist < 1e-4f) return {0,0};
    float speed = min(a.maxSpeed * (dist / slowingRadius), a.maxSpeed);  // ramp down near target
    Vec2 desired = offset * (speed / dist);
    return clampMag(desired - a.velocity, a.maxForce);
}
// Steering writes velocity only (forward Euler on velocity). The physics character
// controller takes a.velocity as its desired velocity and owns the position:
// it moves and collides (collide-and-slide, slopes, step-up) and writes the result.
void applySteering(Agent& a, Vec2 steering, float dt) {
    a.velocity = clampMag(a.velocity + steering * dt, a.maxSpeed);
}

