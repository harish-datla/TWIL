# How Unreal Does It


Open almost any Unreal subsystem's threading code and you'll see either FTaskGraphInterface or UE::Tasks. Unreal has two layered task systems, the old one and the new one, and learning Unreal multithreading in 2026 mostly means understanding which one to reach for.
TaskGraph (the legacy system)

The original Unreal task system, FTaskGraphInterface, is a job graph where each task has prerequisites and runs on either a specific named thread or "any thread." The named threads are GameThread, ActualRenderingThread, RHIThread, AudioThread, and a few others[25]. The reason named threads exist is that many of Unreal's subsystems (UObjects, the RHI that wraps the graphics APIs, the audio engine) are not thread-safe, so work that touches them has to be funneled onto a specific thread.

This works, but it's rigid. Once several non-thread-safe systems each need exclusive access, you either add more named threads or queue everything onto one thread and bottleneck on it. The newer API offers a third option.
UE::Tasks (the new system)

Introduced in UE 5.0 and matured through 5.3+, UE::Tasks lives in Tasks/Task.h. The model is closer to what we built in §10: launch a task, attach prerequisites, the scheduler handles the rest. Its distinctive primitive is the Pipe[9]:
unreal_tasks.cpp · the new API

using namespace UE::Tasks;

// Independent task, no prereqs.
FTask Physics = Launch(UE_SOURCE_LOCATION, [] { DoPhysics(); });

// Dependent task: waits for Physics, then runs.
FTask Culling = Launch(UE_SOURCE_LOCATION, [] { DoCulling(); }, Physics);

// Multiple prereqs.
FTask DrawLists = Launch(UE_SOURCE_LOCATION, [] { BuildDrawLists(); },
                         Prerequisites(Physics, Culling));

// Pipe: tasks launched through the same pipe never run concurrently.
static FPipe WorldPipe{ UE_SOURCE_LOCATION };
WorldPipe.Launch(UE_SOURCE_LOCATION, [Actor1] { TickActor(Actor1); });
WorldPipe.Launch(UE_SOURCE_LOCATION, [Actor2] { TickActor(Actor2); });
// Both run on workers, but only one at a time.

A Pipe is an alternative to named-thread affinity. Instead of saying "this work has to run on the game thread because UWorld isn't thread-safe," you say "these tasks share the UWorld pipe, so never run two at once." Epic's docs note that execution order is usually predictable but launch order is not guaranteed[9]. The scheduler can put successive pipe tasks on different worker threads as long as they don't overlap in time, which spreads load better than nailing everything to one thread.
Other Unreal pieces worth knowing

    FRunnable / FRunnableThread: a single long-lived dedicated thread. Use for streaming workers, network threads, or anything that wants its own execution forever.
    ParallelFor(N, lambda): parallel loop helper built on the task graph. Docs note it uses TaskGraph for "unbalanced tasks and offers better work distribution among threads at the cost of a little bit more synchronization"[26].
    TPromise<T> / TFuture<T>: standard producer/consumer future-promise pair. The producer eventually calls SetValue; the consumer waits on the future.
    Render thread / RHI thread pipelining: the game thread simulates frame N+1 while the render thread builds frame N's commands, and the RHI thread translates those commands into graphics-API calls, trailing the render thread by up to a frame[27].

Where to look in the engine

If you want to read the source: Engine/Source/Runtime/Core/Public/Tasks/ for the new API, Engine/Source/Runtime/Core/Public/Async/TaskGraphInterfaces.h for the legacy one. Alex Stevens's Unreal Fest Gold Coast 2024 talk "How to Benefit from Multithreading in Your Unreal Engine Projects"[28] is a recent walkthrough, with a companion repository of example code.

[Back to Job Systems from Scratch](index.md)
