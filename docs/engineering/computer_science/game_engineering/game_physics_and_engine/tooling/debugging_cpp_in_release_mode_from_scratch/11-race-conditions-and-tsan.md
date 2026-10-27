# Race conditions and TSan

Two job threads each add their counts to a shared total, and once in a while the total comes out short. A data race in C++ is two threads accessing the same memory location, at least one of them writing, at least one of them not atomic, and neither access happening before the other[36]. The C++ memory model declares this to be undefined behavior; the optimizer is permitted to reorder, hoist, or fuse the accesses on the assumption that races don't happen. The Memory Model tutorial covers the model in depth; this section is the debugging side.

The right tool for detecting races is ThreadSanitizer[27]. Each thread carries a vector clock; a release (unlock, release store) publishes the thread's clock and an acquire (lock, acquire load) merges it into the acquiring thread's. Every memory access is recorded in shadow memory with the accessing thread and its clock value, and an access is reported as a race when an earlier conflicting access by another thread isn't ordered before it by those clocks. The instrumentation is heavy (5–15× slowdown), but because it checks ordering rather than timing it flags a race even when this run's interleaving happened to be harmless. It reports no false positives on code whose synchronization it can see; it can still miss races, since it keeps only a bounded history of earlier accesses per location.

Worked example

The counter holds 41. Thread A reads 41. Before A writes, thread B also reads 41, adds one and writes 42. Then A writes its own 41 + 1 = 42. Two increments ran and the counter moved by one. In the widget, Run gives each thread 1,000 increments, so the right answer is 2,000; naive mode usually finishes between about 1,360 and 1,410, with around 600 updates lost. Finding that under TSan costs time: at its typical 5× to 15× slowdown, a 2-minute test takes 10 to 30 minutes, one reason it often runs as its own nightly job.

The widget below is a small simulation. Two threads increment a shared counter. With no synchronization, the result depends on the interleaving. With a mutex or an atomic fetch_add (even with memory_order_relaxed), the count is correct. The right panel shows TSan-style happens-before edges and the race report:
Live · Race detector
naive ++
atomic ++
mutex ++
modenaive
statusidle. press Run or Step.
Run 1000 increments
Step
Reset
Pick a mode, then press Run for 1,000 increments per thread (the right final count is 2,000), or Step to build a two-increments-per-thread run one operation per click. The lanes show the first 16 operations. The naive mode shows the lost-update race: one thread reads, the other reads the same value, both write back the same incremented value, and one increment is lost (red). TSan's verdict doesn't depend on a loss happening: any two conflicting accesses from different threads with no happens-before path between them are a race, so naive mode is reported even on a run that got lucky. Happens-before is TSan's underlying construct: an access happens before a later one if a chain of program-order and synchronization edges connects them[27]. The green edges in mutex mode run from an unlock to the other thread's next lock (same-thread edges are plain program order and aren't drawn). In atomic mode they assume the default seq_cst fetch_add, whose acquire-release semantics link each increment to the next; with memory_order_relaxed there would be no edges, the count would still be exact, and TSan would still report nothing, because a race needs at least one non-atomic access.
When TSan can't help

TSan instruments user-space accesses with the runtime it controls. The bugs it can't catch:

    Signal-handler bugs. A handler runs on the thread it interrupted, so a conflict between the handler and the code it interrupted isn't a cross-thread race, and TSan's happens-before model doesn't flag it. TSan does report async-signal-unsafe calls such as malloc inside a handler; the rest needs review against the POSIX async-signal-safe list.
    Races on memory shared with another process or kernel-side. A driver writing to a buffer your process maps via mmap, a SHM region another process is touching, a GPU DMA. None of these go through TSan's instrumented load/store; TSan sees only your side.
    Races in code TSan didn't instrument. Inline or hand-written assembly, and prebuilt libraries compiled without -fsanitize=thread. Their loads and stores are invisible, and synchronization done inside them can produce false reports in the instrumented code around them.
    Races the test never executes. TSan checks the accesses that actually happen. If the two threads never touch the location during the run, there is nothing to flag; coverage is the limit.

For the bugs TSan can't catch, the next-best tool is rr or TTD: record a failing run, inspect the interleaving deterministically. rr's chaos mode (rr record --chaos) randomizes scheduling decisions to make rare interleavings show up more often under recording. For races that won't reproduce locally, the tactic is to add invariant checks in the suspect region (assert(state == EXPECTED)) and ship the build to a wider set of testers; the assert turns the silent corruption into a defined crash with a stack trace.
