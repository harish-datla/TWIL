# Measure, then prove it

Two engineers argue about whether the particle system or the AI is slowing the game down. Measure, don't guess. Intuition about hotspots is often wrong: profile first, then optimize the actual bottleneck. Amdahl's law caps the payoff: speeding up a function that's 2% of the frame can't help much, so find the part that dominates. Knuth, in full (the truncated version is routinely misused)[25]:

    "We should forget about small efficiencies, say about 97% of the time: premature optimization is the root of all evil. Yet we should not pass up our opportunities in that critical 3%."

Worked example

A function that takes 2% of a 16.7 ms frame costs 0.33 ms. Make it twice as fast and you save 0.17 ms, 1% of the frame. A render pass that takes 40% of the frame costs 6.7 ms; cut its time by a quarter and you save 1.7 ms, 10% of the frame. The profile tells you which one you're looking at before you spend the week.
Profile a representative workload

Profile the busy combat scene, not the title screen: the bottleneck on an empty menu is not the bottleneck in gameplay. And don't blame a frame spike on one cause by reflex: pop-in, an allocation hitch, a fat physics frame, OS preemption, and shader-compile stutter are distinct causes; the profiler tells you which.

Once you've changed something, show that it helped. A 0.3 ms win can be smaller than the gap between two runs of the same build. Google Benchmark's documentation lists the usual sources of that noise: CPU frequency scaling and turbo boost, cores that run at different speeds, the scheduler moving threads around, a hyperthread sibling sharing the core, other programs, and cache effects from code on other cores. On Linux its suggested remedies are the performance frequency governor, turning boost off, and pinning the program to one core[26]. On a game you usually can't remove all of it, so run the same scene several times per build and compare the spreads.

Worked example

Five runs of the same camera flythrough give median frame times of 14.1, 14.4, 14.2, 14.6 and 14.3 ms on the old build and 14.0, 14.3, 14.5, 14.1 and 14.2 ms on the new one. The middle run of each set is 14.3 ms against 14.2 ms, a 0.1 ms difference, while each build's own runs spread over 0.5 ms. These runs don't show the new build is faster. A change that took the new build to 13.2 to 13.6 ms would: its slowest run beats the old build's fastest.
Microbenchmarks and builds

A microbenchmark that times a function in a loop has one more trap: if the result is never used, the optimizer can delete the work and you time an empty loop. Google Benchmark's DoNotOptimize forces a result to be stored in a register or memory, so the compiler can't drop the work as unused, though it can still fold a result it knows at compile time[26]. And profile an optimized build with debug symbols: a debug build spends its time in code the optimizer would have inlined or removed, so its hot spots are often in the wrong places.
