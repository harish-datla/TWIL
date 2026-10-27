# Why Lock-Free


Two concrete cases pull an engine toward lock-free queues, and one honest reason to think twice.

    The job pool. Every worker pulls tasks from a shared queue. Put a mutex on it and the workers serialize on that one lock exactly when you wanted them parallel (the scaling problem from the Job Systems tutorial).
    The audio thread. The audio callback runs under a hard deadline. If it takes a mutex and a lower-priority thread holds it, priority inversion (the high-priority callback stuck waiting on a lock a low-priority thread holds) can stall the callback past its deadline and the sound glitches. The control thread hands parameters to the audio thread through a lock-free ring, a fixed array used as a circular queue, instead[11].
    The counterpoint. Lock-free is genuinely hard to get right and harder to test. Often a mutex plus a ring buffer is fast enough. Reach for lock-free when you can prove the lock is the bottleneck, and prefer it for progress and tail-latency, not raw throughput.

Worked example

Say the audio device runs at 48 kHz with a 256-sample buffer. The callback has to fill each buffer before the device needs it: 256 / 48,000 = 5.3 ms, every 5.3 ms, for as long as the game runs.

If the callback blocks on a mutex while the thread holding it waits to be scheduled, that wait comes out of the same 5.3 ms. Miss it once and the device plays a gap. A lock-free ring removes the wait: pop either finds a command or returns at once.

[Back to Lock-Free Queues and Memory Ordering](index.md)
