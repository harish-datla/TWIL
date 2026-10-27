# Pitfalls
"Immediate mode is faster"
It's about where state lives. The win is iteration speed + no desync, not throughput.
Shipping ImGui as the player HUD
It targets tools and debug UI; its README says full internationalization and accessibility aren't supported.
CPU-timing GPU work
GPU work runs later, so a CPU timer measures recording. Use timestamp queries.
Reading the timestamp the same frame
WAIT_BIT stalls the CPU. Ring the pool, read a few frames old.
32-bit timestamps
Overflow in ~4.3 s at a 1 ns tick. Use 64-bit; mask to timestampValidBits.
Quoting average FPS
It hides hitches. Report frame times: P99 and time over budget.
A zone around every tiny call
Each zone costs tens of ns on the thread it measures. Zone systems and loops.
Sampling without frame pointers
GCC omits them from -O1 up, and perf's default unwinder then gives bogus stacks. Build with -fno-omit-frame-pointer or unwind with DWARF.
Optimizing the widest total
Total includes callees. Look for where self time piles up.
Summing per-pass GPU times
Passes overlap. Time the whole frame with its own timestamp pair.
Reading vsync as GPU-bound
Both idle to the refresh line. Diagnose with vsync off.
Optimizing without profiling
Intuition is often wrong. Measure, then fix the actual bottleneck.
Profiling the title screen
Profile a representative busy workload, not the menu.
One run per build
Run-to-run noise can exceed the win. Compare several runs of the same scene.
Profiling the debug build
Its hot spots move. Profile an optimized build with symbols.
