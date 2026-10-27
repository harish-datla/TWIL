# CPU profiling

A frame that should take 12 ms sometimes takes 21, and you need to know which system ate the other 9. There are two ways to find where CPU time goes, and they're a real tradeoff[7]:

    Sampling: interrupt the program periodically and record the call stack. Low overhead, statistical, but can miss short rare events and gives no exact call counts (perf, VTune).
    Instrumentation: explicit scope markers in code give exact per-scope timing, but you only see what you marked and the markers add overhead (Tracy, Optick).

A sampling profiler needs no changes to your code. The operating system interrupts the program at a fixed rate and records the call stack of the running thread. Linux perf record defaults to 4,000 samples a second[8]; Tracy samples at 8 kHz on Windows and 10 kHz on Linux[9]. To record a stack, the profiler has to find each function's caller. The cheap way follows the chain of saved frame pointers, but GCC omits them by default at -O1 and above[10], and perf's documentation warns that its default frame-pointer unwinding then "will produce bogus call graphs". Build profiling binaries with -fno-omit-frame-pointer, or switch perf to DWARF unwinding with --call-graph dwarf[11].

Sampling is a poll, so its answers carry error bars. If a function really takes a share of the time, the share of samples that land in it scatters around the true value by a predictable amount:
margin ≈ 2 × √( p (1 − p) / n )

In words: the margin shrinks with the square root of the number of samples, so four times as many samples halve it. Small shares get small absolute margins but large relative ones.

Worked example

Physics takes 24% of the frame and you have 400 samples: 2 × √(0.24 × 0.76 / 400) = 0.043, so the profile says 24% ± 4%. At 4,000 samples, one second of perf's default rate on a busy thread, the margin is ±1.4%. A hitch that averages 1.4% of the time gets ±1.2% at 400 samples, a margin nearly as large as the share itself.

Each catches what the other misses, and modern tools blend both (Tracy is "a hybrid frame and sampling profiler")[12]. Tracy's manual describes the split: statistical sampling "may show you the hot spots in your code, but it cannot accurately pinpoint the underlying cause for semi-random frame stutter"[9]. Instrumentation records a named scope's begin and end timestamps into a per-thread buffer, drawn as a flame chart (nested bars on a time axis: width = duration, stacking = call depth), one track per worker thread. The next section builds one.

Worked example

The widget below simulates a frame of about 12.3 ms: render 5.2, physics 3.0, update 2.0, anim 1.2, present 0.5 and input 0.4, plus a 0.9 ms hitch in about one frame in five. A sampler that takes one sample per frame at a random instant lands in physics about 3.0 / 12.5 ≈ 24% of the time and in render about 42%. The hitch averages 0.9 × 0.2 = 0.18 ms a frame, so it gets about 1.4% of samples: one or two in 100, and roughly a 1-in-4 chance of none at all. Instrumentation records it every frame it runs. Turn on the physics spike and the solver's 1.4 ms becomes 9.8, the frame grows to about 21 ms, and physics takes about 55% of the samples.

A simulated frame's scopes as a flame chart. Trigger a spike and the physics scope dominates; compare sampling to exact instrumentation:
instrumentation
sampling
physics spike
You're looking at one frame's scopes, stacked by call depth, against a 16.7 ms budget line. Each frame re-rolls the scope timings; each bar sits on top of the scope that called it. Hit physics spike and the solver balloons, so you can localize the frame spike by eye. Switch to sampling: each update simulates ten frames and takes one sample at a random instant in each, and the tally converges on each system's share of frame time, with a ± margin that narrows as samples pile up. The pink hitch runs in only about one frame in five and for under a millisecond, so it collects few samples or none, while instrumentation shows it every time it runs.
