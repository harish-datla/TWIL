# Engine Tooling: Dev UI & Profiling

Engine tooling turns “the game feels slow” into measured facts. This tutorial builds debug UI, frame-time statistics, CPU and GPU profilers, trace output, profiling workflows, and hot reload.

Time: ~85 min  
Level: Senior  
Prerequisites: The Game Loop, Job Systems, and the GPU Pipeline.  
Stack: C++ & Rust · Vulkan  
Phase: 13 · Tooling

## What you’ll be able to do

- Add Dear ImGui or egui debug panels to a frame loop.
- Read frame times and percentiles instead of relying on FPS alone.
- Build a scoped profiler that writes traces Perfetto can open.
- Time GPU passes with delayed timestamp queries.
- Distinguish CPU-bound, GPU-bound, and vsync-capped frames.
- Prove performance changes with repeated runs and use hot reload to shorten iteration.

## Sections

- [Seeing the Engine](01-seeing-the-engine.md)
- [Immediate vs Retained](02-immediate-vs-retained.md)
- [Dear ImGui & Panels](03-dear-imgui-and-panels.md)
- [Frame Time, Not FPS](04-frame-time-not-fps.md)
- [CPU Profiling](05-cpu-profiling.md)
- [A Tiny Profiler](06-a-tiny-profiler.md)
- [Reading a Profile](07-reading-a-profile.md)
- [From Where to Why](08-from-where-to-why.md)
- [GPU Profiling](09-gpu-profiling.md)
- [CPU- vs GPU-Bound](10-cpu-vs-gpu-bound.md)
- [Measure, Then Prove It](11-measure-then-prove-it.md)
- [Hot Reload](12-hot-reload.md)
- [Pitfalls](13-pitfalls.md)
- [What’s Next](14-whats-next.md)
