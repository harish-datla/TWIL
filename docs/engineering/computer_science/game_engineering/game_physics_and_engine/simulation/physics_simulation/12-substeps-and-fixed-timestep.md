# Substeps and fixed timestep

Render frame times vary from one frame to the next, and a solver tuned for one timestep behaves differently at another. Physics usually runs at a fixed timestep, for reproducibility and because solver tuning and stability limits depend on dt. The standard pattern, popularized by Glenn Fiedler's "Fix Your Timestep!"[13]: an accumulator absorbs real elapsed time, and the physics loop ticks in fixed-size bites.

Worked example

A 144 Hz display renders every 6.94 ms, and physics steps at 1/120 s, 8.33 ms. The first frame's accumulator holds 6.94 ms, not enough for a step, so it runs none and renders with alpha = 0.83. Each of the next five frames runs one step. Over those six frames (41.7 ms) physics runs five 8.33 ms steps, and the render interpolation hides the frame that ran none.

// Fixed-timestep accumulator pattern.
// 'elapsed' is wall-clock time since last render frame (variable).
// 'physDt' is the fixed physics timestep (e.g. 1/120 s).
accumulator += elapsed;
while (accumulator >= physDt) {
    stepPhysics(physDt);
    accumulator -= physDt;
}
// interpolation factor for rendering between the last two physics states
float alpha = accumulator / physDt;
renderState = lerp(previousPhysicsState, currentPhysicsState, alpha);

The rendering interpolation (the alpha lerp) smooths visual motion between discrete physics ticks. Without it, objects stutter whenever the render rate and the physics rate do not line up, because some frames get two physics steps and others none.

Variable dt breaks determinism. A variable dt comes from measured frame times, which differ from run to run and machine to machine, so the same inputs step through different sequences of dt values. Those sequences do not land on the same state: the integrator's error depends on the step size, and floating-point rounding means even one 1/30 s step and two 1/60 s steps disagree. Replays desync, networked physics diverges, and QA cannot reproduce bugs. A fixed dt is a prerequisite for reproducibility, though not sufficient on its own (see the floating-point pitfall in §15).

Substepping (running multiple physics ticks per frame with a smaller dt) improves stability for stiff constraints and high-speed objects. Jolt's PhysicsSystem::Update takes a number of collision steps that splits each update into smaller collision-plus-integration steps, and its documentation says the system is generally stable at 60 Hz with one[9]. PhysX's TGS solver substeps internally: it splits the step into as many substeps as there are position iterations and integrates after each, which its documentation calls conceptually similar to running PGS several times at a smaller timestep with one iteration each[12].
