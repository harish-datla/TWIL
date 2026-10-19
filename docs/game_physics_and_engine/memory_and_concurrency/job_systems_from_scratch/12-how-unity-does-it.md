# How Unity Does It


If you write gameplay code in Unity, you'll meet these ideas through its C# job system rather than through a scheduler of your own. Unity's C# job system, released in 2018, is a thin managed layer over the native job system Unity uses for its own engine code[8]. Three pieces matter to a user of it.
The job interfaces

You implement a job by writing a struct with one method. The struct is the body; the fields are the inputs and outputs. Unity copies the struct into the scheduler, so jobs can't capture references the way a closure would.

    IJob: one unit of work, runs once on a worker.
    IJobParallelFor: N independent iterations; the scheduler picks a batch size and hands out ranges to workers.
    IJobFor: like IJobParallelFor, but the same job can run on the calling thread, sequentially on one worker, or in parallel batches.
    IJobChunk / IJobEntity: ECS-specific; iterates over archetype chunks. Used inside DOTS / Entities packages.

Particles.cs · a Unity IJobParallelFor

using Unity.Burst;
using Unity.Collections;
using Unity.Jobs;
using Unity.Mathematics;

[BurstCompile]
struct UpdateParticles : IJobParallelFor {
    public NativeArray<float3> positions;
    [ReadOnly] public NativeArray<float3> velocities;
    public float dt;

    public void Execute(int index) {
        positions[index] += velocities[index] * dt;
    }
}

// At call site:
var job   = new UpdateParticles { positions = pos, velocities = vel, dt = 0.016f };
var handle = job.Schedule(pos.Length, 64);   // one Execute(index) per particle, batchSize=64
handle.Complete();                          // block until done; call it as late as possible

Three things to notice

1. Handles are dependencies. Schedule returns a JobHandle. Pass it into the next Schedule call to express "this depends on that." JobHandle.CombineDependencies(a, b, c) gives you a handle that's satisfied when all three are. The result is the same DAG you'd build with our prereqs array[8].

2. The safety system catches data races at schedule time. Every NativeArray tracks which jobs read or write it. If you try to schedule a job that writes to a container that another scheduled job is also writing, Unity throws an exception with a clear message before either job runs. The AtomicSafetyHandle checks are compiled in only when ENABLE_UNITY_COLLECTIONS_CHECKS is defined, which by default means the Editor, so player builds skip them[8]. For programmers new to multithreading this is the most useful part of the system, because it turns a race that might corrupt data silently into an exception at schedule time.

3. Burst is a separate piece. [BurstCompile] hands the job's struct to Unity's Burst compiler, which translates a restricted dialect of C# called HPC# (high-performance C#) through LLVM into target-specific machine code, vectorized where the loop allows[24]. The Unity.Mathematics vector types (float4, float3, int4) are designed to map onto SIMD registers. The job system and Burst are orthogonal: you can use jobs without Burst, and you can Burst-compile functions that aren't jobs. In Unity 6.6 Burst moves from a package into the engine as built-in modules[24].
Awaitable is not the job system

Unity 6 added an Awaitable type with async/await integration. It's for main-thread sequencing ("wait until next frame", "wait for a download to finish"), not for parallel compute. The docs are explicit: "To get the most of multi-core CPUs and parallelize your algorithms, use the job system instead"[43]. The two live at different layers, and a game can use both.

[Back to Job Systems from Scratch](index.md)
