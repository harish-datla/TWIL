# Sequentially Consistent: The Strong Default


A worker thread that runs out of jobs sets sleeping = true, checks the job count one last time, and goes to sleep if it's zero. A producer adds a job, then checks sleeping to decide whether to wake the worker. That's §1's store-then-load shape on both sides, and acquire/release doesn't make it safe. The ordering that does is seq_cst, the default for std::atomic operations and the strongest and most expensive ordering. The promise: every seq_cst operation, across every thread, fits in a single total order that all threads agree on. That agreement is what forbids the SB (0, 0) outcome and IRIW. Acquire/release alone lets each thread's store sit behind its later load, and lets two readers see two writers' stores in different orders; seq_cst on those operations doesn't.

Worked example

Worker: sleeping.store(true), then jobCount.load(). Producer: jobCount.store(1), then sleeping.load(). Both start at 0 / false.

Release stores, acquire loads: both stores can still be in their store buffers when both loads run. The worker reads jobCount = 0 and sleeps; the producer reads sleeping = false and doesn't wake it. A job sits in the queue with nobody running it.

All four seq_cst: the four operations fall into one order both threads agree on, and whichever load comes last in it comes after both stores (the same argument as §4's six sequences). So the worker sees 1 and stays awake, or the producer sees true and wakes it, or both.

What that costs depends on the architecture. On x86, a seq_cst load is a plain MOV, but a seq_cst store is an XCHG (GCC 14 and Clang both emit it; older GCC used MOV + MFENCE), which drains the store buffer, where a release store is a plain MOV. On ARMv8, seq_cst loads and stores compile to the same LDAR and STLR as acquire and release, and that's enough for seq_cst because an LDAR can't complete ahead of an earlier STLR and ARMv8 is multicopy atomic[35]. The cost there is that stall: a load right after a store waits for the store to drain. Targets with ARMv8.3's LDAPR avoid it for plain acquire loads, which compilers emit when you build for those cores. On ARMv7, which has neither instruction, every seq_cst access carries full DMB barriers. P0668[11] didn't change any of these instruction sequences; it changed the standard's wording so that the existing POWER and ARMv7 mappings conform.

Rules of thumb:

    Use seq_cst by default while you're learning. It's the easiest to reason about, and on the algorithms you're learning with the cost rarely matters.
    Use acquire/release in production code where the pattern is publication. On x86 a release store is a plain MOV instead of an XCHG; on ARMv8.3+ targets an acquire load can be an LDAPR that doesn't wait for earlier release stores.
    Use relaxed for counters and statistics. Hit counts, frame numbers, profiler markers. No ordering is needed; you only want atomicity and a single modification order.
    Don't use consume. Compilers treat it as acquire, and C++26 deprecates it.

A common overspend: a counter incremented by many threads and read now and then by one. Neither the increments nor the reads need seq_cst; a relaxed RMW is correct, and on ARM it drops the acquire and release halves of the instruction. Spend ordering only on the atomic whose value tells another thread it can now read something else.

[Back to The C++ Memory Model from Scratch](index.md)
