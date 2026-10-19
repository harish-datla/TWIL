# Scheduling and parallelism

A frame has twenty systems and eight cores, and running them one after another leaves seven cores idle. Each system declares which component types it reads and which it writes. Two systems can run in parallel if they have no write-write or read-write conflict on the same component type. A system that reads Position and writes Velocity can run in parallel with a system that reads Health and writes Damage, because the component sets are disjoint.

The scheduler builds a dependency graph of systems. Edges encode conflicts: if system A writes Position and system B reads Position, B depends on A (or vice versa, depending on declared ordering). The scheduler topologically sorts this graph and dispatches independent systems to worker threads. Bevy's multi-threaded executor[18] does this automatically each frame. The developer writes systems with declared access; the engine parallelizes them.

This only works because ECS access is declarative. In an OOP architecture, a method on a GameObject can touch any field on any other object through a pointer. The engine has no way to know what a method accesses without running it. In ECS, the query signature is the access declaration, and the scheduler reads it when the schedule is built.

// Bevy system declarations (Rust). The scheduler reads the type signature
// to determine access: &Position asks for read access, &mut Velocity for write.
// Written against Bevy 0.17+, where buffered events were renamed to messages
// (EventReader -> MessageReader) and Time::delta_seconds became delta_secs.

fn movement_system(
    mut query: Query<(&mut Position, &Velocity)>,
    time: Res<Time>,
) {
    for (mut position, velocity) in &mut query {
        position.x += velocity.dx * time.delta_secs();
        position.y += velocity.dy * time.delta_secs();
    }
}

fn damage_system(
    mut query: Query<&mut Health, With<DamageReceiver>>,
    mut damage_messages: MessageReader<DamageMessage>,
) {
    // Reads DamageMessage, writes Health. No overlap with movement_system,
    // so the scheduler runs both in parallel. The reader must be `mut`:
    // iterating it advances a per-system cursor into the message buffer.
}

// Registration: the access sets come from the parameter types; the schedule
// compares them for conflicts when it is built, before the first frame runs.
app.add_systems(Update, (movement_system, damage_system));

Live · System dependency scheduler
Click a cell to cycle: no access → read → write
Pos
Vel
HP
AI
Render
movement 3ms
W
R
·
·
·
physics 4ms
W
W
·
·
·
damage 2ms
·
·
W
·
·
ai 5ms
R
·
R
W
·
cull 2ms
R
·
·
·
W
animate 3ms
·
·
·
·
W
Worker threads
1
2
3
4
reset access
conflict edges
7
longest chain
12 ms
frame
12 ms
core utilization
53%
A matrix of systems against components, where each cell cycles through no access, read and write, with the resulting schedule drawn as one lane per worker below it. Edit cells and change the worker count. Two systems conflict on a component if either one writes it. Read-read is not a conflict, and that is the entire reason declaring access buys parallelism. Every cell edit rebuilds the conflict graph, retakes the longest weighted chain through it (red), and repacks the systems greedily onto the lanes; nothing here is precomputed. At the default access sets the chain is movement → physics → ai, 12 ms, and it is a floor: two workers already reach it, and three or four move only the utilization number. Clear physics's Pos access and set its Vel to read: seven conflict edges drop to four and the chain drops to 8 ms, because Pos and Vel were the only two components coupling it to movement.

Two engines read the same declarations and stop at different guarantees. Unity's Entities package chains job dependencies from declared component access and is explicit about the granularity it settles for: "this system dependency approach works at a system level, it can result in jobs waiting for other jobs to access components that the original jobs don't need."[10] Bevy's multi-threaded executor promises exclusion and nothing more. Its schedule module describes the executor as one where "Non-conflicting systems can run in parallel,"[30] and a conflicting pair with no .before or .after between them is an ambiguity, defined in the build settings as "systems with conflicting access but indeterminate order."[31] Ambiguity detection is off by default. The widget models the Unity reading, where declaration order fixes the direction of every conflict edge and the graph is a DAG by construction. Flecs takes a third position and pins work to threads rather than to systems: "The scheduler ensures that the same entity is always processed by the same thread, until the next sync point"[17], with the sync points themselves derived from the read/write patterns across systems.
Check Your Understanding Q1 of 3

System A writes Position. System B reads Position and writes Velocity. System C reads Health. Which pairs can run in parallel?
A and B only.
A and C, and B and C.
All three can run simultaneously.
0/3 correct
