# Update order & the AI stack

The player presses attack this step. If AI runs before input is applied, the enemies decide without knowing the attack started, and react a step late. The order inside simulate is one common choice, and each edge has a reason: input precedes AI (AI reacts to this step's commands), AI precedes gameplay (decisions set intents gameplay executes), gameplay precedes physics (physics resolves the requested motion into authoritative post-collision transforms), and animation follows physics (poses match the motion that actually happened). The subtlety is consistency: an object that reads a peer's transform partway through the update sees some peers already stepped and some not, the bug Gregory calls a one-frame-off lag[1]. Reading from the previous step's snapshot, or phasing the update so every read comes after the writes it depends on, avoids it.

Worked example

Enemy A updates before enemy B, and each aims at the other, both moving at 6 m/s with 60 steps a second. A reads B's position from the previous step, 6 / 60 = 0.1 m out of date, while B reads A's position from this step. Swap their order in the entity list and the error swaps with it. Reading every position from last step's snapshot makes both 0.1 m stale, the same for everyone, whatever the order.
The AI stack is four layers; don't conflate them

"The enemy moves toward the player" is four different modules: the behavior tree decides (returns SUCCESS/FAILURE/RUNNING), the navmesh + funnel plans the path (requestPath, stringPull), steering produces motion (arrive, clamped to maxForce/maxSpeed), and the character controller resolves that motion against geometry (collide-and-slide). Pathfinding is global and discrete; steering is local and continuous; they're not the same function.
One enemy through the four-layer AI stack
C++
Rust

// Runs inside aiTick (step 2): DECIDE, PLAN, MOVE. Writes a desired velocity, never a position.
void updateEnemyAI(Enemy& enemy, float dt) {
    enemy.behaviorTree->tick();                        // DECIDE: the MoveTo leaf (built with refs to enemy.blackboard
                                                       //   and enemy.navAgent) calls navAgent.requestPath(target)
    Vec2 steering = {0.0f, 0.0f};
    if (enemy.navAgent.hasPath()) {                    // PLAN: the funnel's corners on the floor plane (hasPath and
        Vec2 corner = enemy.navAgent.currentCorner();  //   currentCorner are read-only accessors added to NavAgent)
        steering = arrive(enemy.agent, corner, enemy.slowingRadius);   // MOVE: local force, clamped to maxForce
    } else {
        steering = clampMag(enemy.agent.velocity * (-1.0f / dt), enemy.agent.maxForce);  // no path: brake toward rest
    }
    applySteering(enemy.agent, steering, dt);   // velocity only: the character controller owns position (step 4)
}

// Runs inside physicsStep (step 4): RESOLVE the requested motion against the level.
void resolveEnemy(Enemy& enemy, const World& world, float dt) {
    Vec3 desired = {enemy.agent.velocity.x, 0.0f, enemy.agent.velocity.y};        // navmesh plane (x, z) -> world, y up
    Vec3 resolved = enemy.controller.moveAndSlide(desired, world.collision, dt);  // capsule collide-and-slide
    enemy.agent.position = {resolved.x, resolved.z};  // next tick steers from where the capsule actually stopped
}

The character controller is the one piece with no prior tutorial: a thin kinematic capsule that collide-and-slides against the collision shapes the physics module built, added here as glue. The split into two functions follows the update order: AI writes only a desired velocity in step 2, and the controller, in the physics step, is the only thing that writes position. arrive, clampMag and applySteering are the steering module's; applySteering writes velocity only, which is what lets the controller own position.
