# Job Systems from Scratch

A frame needs 24 ms of CPU work, the budget at 60 fps is 16.6 ms, and the machine has eight cores. Giving rendering, physics and gameplay a thread each helps until one of them becomes the bottleneck, and then the other cores sit idle. The fix shipping engines use is a job system: split each frame's work into small jobs and let every core take them. This page builds one, starting from one thread and a locked queue and fixing its problems one at a time, up to a work-stealing scheduler with dependency tracking in C++ and Rust and the fibers Naughty Dog layered on top for The Last of Us Remastered. It needs no engine or framework, only the standard library, and each major idea has a live demo.
Time
~60 min
Level
Junior engine programmer
Prereqs
You can read C++ or Rust at a beginner level
Hardware
Some passing acquaintance with the idea that CPUs have cores and caches helps
◂ Build a Game Engine
Phase 1 · Memory & concurrency
Next · SIMD ▸
What you'll be able to do by the end

    Explain why one thread per subsystem stops scaling, and size a worker pool for a machine.
    Spot the three things that slow a naïve worker pool down as cores are added (lock contention, cache-line ping-pong, no way to express dependencies) and fix each one.
    Read and write a work-stealing scheduler with dependency tracking, and pick a job size that keeps queue overhead small.
    Work out a frame's best possible time from its total work and its longest dependency chain.
    Explain what fibers buy Naughty Dog's engine, how Unity's and Unreal's task APIs map onto the same ideas, and recognize ABA, priority inversion, memory-ordering and SMT bugs in a scheduler.


## Sections

- [Why a Game Engine Needs a Job System](01-why-a-game-engine-needs-a-job-system.md)
- [A Short History of Getting Parallel](02-a-short-history-of-getting-parallel.md)
- [The Naive Worker Pool](03-the-naive-worker-pool.md)
- [Threads versus Tasks](04-threads-versus-tasks.md)
- [Cache Lines and the False-Sharing Trap](05-cache-lines-and-the-false-sharing-trap.md)
- [Lock-Free Queues](06-lock-free-queues.md)
- [Work Stealing](07-work-stealing.md)
- [Dependencies as a DAG](08-dependencies-as-a-dag.md)
- [Fibers and Why Naughty Dog Uses Them](09-fibers-and-why-naughty-dog-uses-them.md)
- [A Working Scheduler, in Your Language](10-a-working-scheduler-in-your-language.md)
- [Try It Yourself](11-try-it-yourself.md)
- [How Unity Does It](12-how-unity-does-it.md)
- [How Unreal Does It](13-how-unreal-does-it.md)
- [The 2026 State of the Art](14-the-2026-state-of-the-art.md)
- [Pitfalls and How to Spot Them](15-pitfalls-and-how-to-spot-them.md)
- [Where to Go from Here](16-where-to-go-from-here.md)
- [Sources & Further Reading](17-sources-and-further-reading.md)
