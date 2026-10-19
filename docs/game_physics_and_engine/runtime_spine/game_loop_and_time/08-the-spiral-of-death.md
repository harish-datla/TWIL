# The Spiral of Death


An explosion spawns two hundred rigid bodies, and one physics step now takes 20 ms to compute against a 16.7 ms step. One failure mode of the accumulator deserves its own name, the spiral of death. If a single simulation step costs more than dt of real time to compute, each frame adds more to the accumulator than the inner loop can drain, so next frame needs even more steps, which take even longer[3]. The loop spirals until it locks up.

Worked example

Take dt = 16.7 ms and a step that costs 20 ms, and assume a frame takes exactly as long as its steps. One step makes the frame 20 ms long, which feeds 20 ms into the accumulator: one step again, with 3.3 ms left over. The leftover grows by 3.3 ms a frame until a frame needs two steps, which take 40 ms, which feed in 40 ms, and so on. Steps per frame go 1, 1, 1, 1, 1, 2, 2, 2, 3, 3, 4, 5, 6, 7, 8, 10, 12, 14, 17, 21, and keep compounding.

Cap the loop at 5 steps and drop the leftover, and every frame takes 5 × 20 = 100 ms to simulate 5 × 16.7 = 83 ms. The game runs at 83% speed instead of freezing.

There are two guards, working from opposite ends, and engines use one or both:

    Clamp the incoming frame time (the frameTime > 0.25 line). Caps how much work can enter the accumulator per frame.
    Cap the steps per frame. Bail out of the inner loop after a maximum count; "the game will slow down then, but that's better than locking up completely"[2]. When the cap trips, drop the leftover time as well, or the accumulator keeps the debt and grows every frame. Unity's Maximum Allowed Timestep caps the simulated time per frame in the same spirit[7].

The guard trades accuracy for liveness

Neither guard makes the simulation keep real time when the machine can't. They make it fall behind in slow motion instead of freezing. The runaway needs a per-step cost near or above dt, and several things can push a loop there or kick it off (a too-small dt, an expensive collision frame, a GC or allocation hitch, OS preemption), so there is no single fix, just the two backstops.

The widget models that loop, with a frame taking exactly as long as its steps:
Sim cost / step 8 ms
clamp + cap steps
The top lines read out the 16.7 ms step budget, the per-step cost, the steps this frame and the leftover accumulator; the plot tracks steps per frame (orange) and accumulator depth (violet). At the default 8 ms per step it takes one step a frame with an empty accumulator. Drag the cost to 20 ms with the clamp off and steps per frame climb every frame, past 10 within about half a second and up to the demo's 100-step safety limit within about a second; from then on the accumulator grows by about a third of a second every frame. That's the runaway. Tick "clamp + cap steps" and the readout settles at "steps this frame: 5 (capped, leftover dropped)" with the accumulator at 3 ms: the simulation slows down, but the loop stays alive.

[Back to The Game Loop & Time](index.md)
