# Work Stealing


Sixteen workers popping from one shared queue spend much of their time contending for its head and tail. A single global queue, even a lock-free one, is still one contention point for N workers. Bitsquid shipped exactly that and argued it was fine at their granularity (at most 5 × thread-count tasks per heavy job) until core counts passed about 32[41]. As jobs get smaller and cores more numerous, the alternative is a queue per worker.

That works fine when the work is balanced. It falls apart the moment one worker finishes early and the others are still backed up: worker A is idle, worker B has a queue of fifty jobs, and they can't help each other. We need a way for idle workers to pull work from busy ones.

Work stealing, the strategy Cilk (an MIT extension of C for parallel programming) put on a provable footing in 1995[5], solves this by making the per-worker queue a deque with asymmetric access:

    The owner pushes and pops at the bottom of its deque. This is the fast path; almost always uncontended.
    A thief (another worker that ran out of work) steals from the top. This is the slow path; contention here is rare but it still has to be correct.

Owner operations are LIFO (last-in, first-out), which keeps a worker's hot data in its L1. Thief operations are FIFO (first-in, first-out), which tends to grab older, larger jobs that are more likely to contain further sub-work for the thief to subsume. The standard lock-free implementation is the Chase-Lev deque[6]. The original paper assumes sequential consistency; Lê et al. (2013) give C11 memory orderings for it, proven correct on ARM and POWER, and that is the version to implement[18].

The widget shows the steals as they happen:
Live · Work Stealing
workers 6
initial imbalance 0.70
Run ▶
Pause ⏸
Reset ⟲
jobs done
0
steals
0
idle workers
0
Each column is one worker's deque, owner's end at the bottom; arrows are steals. Start with one worker holding most of the work. The others fan out, picking random victims to steal from, and the queues converge.
The "work-first" principle

Frigo, Leiserson, and Randall's PLDI 1998 paper on the Cilk-5 scheduler[19] articulates the design principle that makes work-stealing schedulers fast, not just correct: optimize the path where no stealing happens. They call this the work-first principle. Since steals are rare in a well-balanced computation, the common case is "owner pops a job from its own deque and runs it." That path should cost something close to a function call.

Concretely, in our scheduler:

    The owner's push is a plain store into the slot plus a release store of the bottom index; no read-modify-write. Its pop writes the bottom index, then needs one full seq_cst fence before reading the top (store-then-load is the one reordering even x86 allows), and a CAS only when it races a thief for the last element.
    The thief's steal is a single CAS on the top counter.
    The deque storage is owned by one worker (no cross-thread allocation in the fast path).

An uncontended push costs about as much as a couple of ordinary stores; an uncontended pop adds the full fence, on the order of tens of cycles on x86. A steal costs more because its CAS has to pull the top counter's cache line from another core, typically on the order of a hundred cycles or more. As long as steals are a small fraction of operations, the fast path dominates the average. Reinalter's Molecular Matters series[7] walks through a complete C++ implementation with timings; it's worth reading after this section.
How granular is granular?

One question that comes up every time you build a work-stealing scheduler: how big should each job be? Too small and you spend more time managing the queue than running the work. Too big and you can't load-balance because there aren't enough jobs to steal.

The Cilk-5 paper's design principle is that the spawn fast-path should cost only a small multiple of an ordinary function call so that spawn overhead doesn't dominate the work. The practical version: each job should do at least one to two orders of magnitude more work than your queue operations take. If push/pop is around 10 ns, aim for at least 1 µs of work per job. At that ratio, a 16.6 ms frame can schedule on the order of ten thousand jobs without spending serious time on the queue itself; Naughty Dog reported roughly 800 to 1,000 jobs per frame in The Last of Us Remastered[2]. Unity's documentation points the same way: a batchSize of 32 to 128 for cheap per-element work, and as low as 1 when each element is expensive[8].

Worked example

Say a push and a pop cost about 10 ns each, so every job carries 20 ns of queue work.

A job that does 1 µs of real work spends 20 / 1,020 ≈ 2% of its time on the queue. At 100 ns of work it's 20 / 120 ≈ 17%, and at 20 ns it's half.

Naughty Dog's 800 to 1,000 jobs per frame sit far on the safe side: 1,000 × 20 ns = 20 µs of queue work spread over the whole frame.
Check Your Understanding Q1 of 2

Why does the owner of a work-stealing deque push and pop at the bottom while thieves steal from the top?
It’s an aesthetic choice that doesn’t affect correctness.
The owner works alone at its end, so its push needs no read-modify-write and its pop needs a CAS only for the last task. A thief on the top must CAS once per steal because multiple thieves can race, but steals are rare, so the cost is amortized.
Cache prefetchers only work in one direction.
0/2 correct

[Back to Job Systems from Scratch](index.md)
