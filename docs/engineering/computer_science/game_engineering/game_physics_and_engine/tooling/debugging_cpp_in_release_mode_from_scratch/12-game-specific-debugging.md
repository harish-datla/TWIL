# Game-specific debugging

Everything above applies to game engines, and games add three problems of their own. The GPU runs in parallel with the CPU and can hang on its own. The frame budget is hard, so a 4 ms spike is a bug even when nothing on screen is wrong. And a multiplayer bug may live in the divergence between two clients, not in either one alone.
GPU hangs and TDR

A player reports that the screen froze for about two seconds, went black, and the game quit. That's what a GPU timeout typically looks like from the player's side. A GPU command that doesn't complete within the OS-defined timeout (about 2 seconds on Windows[37]) triggers Timeout Detection and Recovery: the OS resets the GPU, the application loses its device, and the next call returns DXGI_ERROR_DEVICE_REMOVED on Direct3D or VK_ERROR_DEVICE_LOST on Vulkan. The CPU-side crash report only shows where the CPU noticed. GPU-side context comes from the runtime when it supports it: D3D12's DRED records breadcrumbs of which GPU operations completed and the faulting address, and Vulkan's VK_EXT_device_fault reports fault addresses and vendor data.

The first tools for a TDR are those breadcrumbs plus the vendor crash-dump tools (NVIDIA Nsight Aftermath, AMD Radeon GPU Detective), which record the command that was in flight when the GPU faulted or hung. A frame capture helps once the hang reproduces: RenderDoc[38] captures every command submitted in a frame and can replay it, so you can bisect the frame's draws and dispatches to find the one that hangs. PIX[39] on Windows does the same with deeper integration into D3D12 and the Xbox toolchain. The shader debuggers in both let you step through a single invocation of a shader.
Frame spikes

Every few seconds the camera stutters, and the frame-time graph shows one 18 ms frame among 14 ms ones. Frame-time anomalies are easier to debug as a profile than as a traditional bug. A 4-ms spike on a 16-ms frame budget is a regression even if no functional output is wrong. The two kinds of profiler:

    Sampling profilers. perf on Linux[40], ETW (Windows Performance Recorder, xperf) on Windows[41], and Superluminal, which samples at 8 kHz on Windows and merges optional instrumentation markers into the same view. Interrupt the program at a fixed rate (ETW defaults to 1 kHz, perf record to 4 kHz), record the call stack, aggregate. Statistical, no code changes, low overhead. Best for "where is the time going overall"; a short spike between samples can go unrecorded.
    Tracing profilers. Tracy[42] (which can also sample), the engine's own scope-based timer, and vendor markers (PIX events feed PIX, NVTX ranges feed Nsight Systems). Record an entry-and-exit timestamp around each annotated region. A small cost per zone rather than zero, but every spike inside an instrumented region is captured and attributable.

A practical default for engine work: automated performance runs on CI to catch broad regressions, instrumented tracing turned on in playtest builds to catch individual spikes. Look at the profiles as a regular practice, not only when something is broken; flame graphs[43] are a compact way to read the sampled ones.

Worked example

At 60 fps the budget is 16.7 ms. A frame that normally takes 14 ms plus a 4 ms spike takes 18 ms, misses the refresh, and with plain vsync is shown one refresh late, 33.3 ms after the previous one: the stutter the player sees. A sampling profiler at ETW's default 1 kHz takes a sample every millisecond, so it lands about 4 samples inside that spike, and Superluminal at 8 kHz about 32. A 0.5 ms spike gets zero or one sample at 1 kHz, which is where tracing zones earn their cost.
Determinism as a debugging tool

Two clients desync 1,024 frames into a match, and never on your machine. A deterministic engine is one that, given the same starting state and the same input log, produces bit-exact identical output. Achieving determinism is non-trivial (no walltime, no rand() without a seeded RNG, no platform-specific floating-point reassociation, no thread-order-dependent updates) but is usually worth doing for the simulation tier of the engine. The payoff for debugging is large: a bug reported as a frame-1024 desync between two clients can be reproduced by replaying the input log; a regression introduced by a refactor is found by running the same log against the old and new binaries and looking at the first frame they differ.

Lockstep multiplayer games, where every client runs the full simulation and only inputs cross the network, are deterministic by necessity. The same machinery doubles as the bug-reproduction tool: the input log from a desynced session is the smallest possible reproduction. Many RTS and fighting games save each match's input stream as the replay file, which serves players and debugging alike.

Worked example

A 20-minute match at a 60 Hz tick is 20 × 60 × 60 = 72,000 frames. Logging an 8-byte hash of the simulation state every frame costs 72,000 × 8 = 576,000 bytes, about 0.6 MB per client. Line up two clients' logs and the first mismatched hash names the frame where they diverged; replay the input log to that frame and compare the two states field by field.
