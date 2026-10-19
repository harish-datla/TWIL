# Why a Game Engine Needs a Job System


A 60 fps game has 16.6 milliseconds to update the world, simulate physics, run animation, cull the scene, build a few thousand draw calls, and stream textures off disk. The PS5 and Xbox Series have eight CPU cores; a gaming PC commonly has six to sixteen. Put all of that work on one core and it overflows the 16.6 ms budget, so the game drops to 30 fps while the other cores sit idle. The job of a job system is to spread that work across every core the machine has.

Spreading the work across cores is harder than it sounds: threads have to coordinate, memory access has to stay cache-friendly, and a naïve solution that hands work to other threads can run slower than the single-threaded version it replaced. A job system is the layer that handles that for the rest of the engine.

The scheduler this page builds is about 200 lines in C++ and in Rust, short enough to read in one sitting. A JavaScript model of it runs live on the page, drives the worker timelines, and powers a playground where you build your own task graphs.
The fixed-thread era is over

In the mid-2000s, when consoles first shipped with multiple cores, engines were typically refactored along subsystem lines. One thread for the renderer, one for physics, one for audio, one for everything else. Bungie's Halo: Reach worked this way, with unique threads for rendering, audio and simulation[3]. It is intuitive because it maps onto the way the engine is already organized. It also stops scaling once every subsystem has a thread, because there is no next subsystem to assign. Worse, the load on each subsystem shifts from frame to frame: a frame heavy on physics leaves the render thread idle, a frame heavy on rendering leaves the physics thread idle, and each frame takes as long as its busiest thread. Bungie's own summary was that static per-thread load balancing "meant suboptimal workload distribution"[3].

The breakthrough, which the industry settled on between roughly 2009 and 2015, was to stop splitting by subsystem and start splitting by data. Instead of one fat physics thread that integrates every rigid body, you have a thousand small jobs, each of which integrates ten bodies. Throw all thousand jobs at a pool of workers, let the workers grab them as they finish whatever they were doing, and every core stays busy. The abstract of Tatarchuk's GDC 2015 talk on the Destiny renderer states it directly: "to optimally take advantage of all available resources, a game engine must be designed from the ground up for job-based multithreading"[3].

The widget below contrasts the two models on a synthetic frame of 24 ms of total work. The "fixed threads" mode runs a render thread, a physics thread, and a gameplay thread, each with its own fixed workload. The "job system" mode splits the same work into small jobs for a pool of workers.

Worked example

The widget's default frame on 8 cores: 24 ms of work, split 12.6 ms render, 7.8 ms physics and 3.6 ms gameplay.

Fixed threads: the frame ends when the slowest thread does, at 12.6 ms. Eight cores were available for 12.6 ms, 100.8 core-ms, and only 24 of them did work: 24% utilization.

Job system: the best possible is 24 ms / 8 cores = 3 ms. Splitting the work into 56 small jobs gets close: the widget reports about 3.1 to 3.3 ms at about 95% utilization, a 4× shorter frame from the same work.
Live · Fixed Threads vs Job System
fixed threads
job system
cores 8
workload skew (heavy physics frame ⇆ heavy render frame) 0.50
RUN FRAME ▶
Reset ⟲
frame time
12.6 ms
core utilization
24%
Each row is a core and each colored block a piece of work, with time running left to right. Press RUN FRAME in each mode and compare the frame time and utilization readouts; then drag the skew slider: the fixed-thread frame follows whichever subsystem is heaviest. Same 24 ms of work, two scheduling strategies. The fixed-thread mode finishes when the slowest subsystem finishes, and extra cores sit idle. The job system finishes when the last job finishes, which is much earlier here because every core helps. The job durations carry a little random spread, so the job-system time shifts slightly each time it is rebuilt.
Check Your Understanding Q1 of 2

Your game runs at 60 fps on a quad-core machine because the renderer fills its thread. You move to an octa-core machine and the framerate stays at 60 fps. Why might that be, even though you have twice the cores?
More cores always means lower frame rate.
Your renderer is still the bottleneck and the four new cores have nothing to do.
The CPU clocks down on octa-core machines.
0/2 correct

[Back to Job Systems from Scratch](index.md)
