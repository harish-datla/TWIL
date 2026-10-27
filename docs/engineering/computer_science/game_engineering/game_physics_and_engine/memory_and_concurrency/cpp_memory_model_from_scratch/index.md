# The C++ Memory Model from Scratch

A loading thread fills a buffer and sets a ready flag, and the render thread sees the flag but reads half-written data, only on the ARM build. Two threads each set a flag and then check the other's, and both see zero. A stats counter gets slower with every worker thread you add. The first two happen because CPUs and compilers reorder memory accesses, the third because cores pass cache lines back and forth. The C++ memory model is the set of rules that says which reorderings another thread can see, and std::atomic's orderings are how you forbid the ones that break your code. This page builds it up from one cache line to acquire/release, a lock-free queue, hazard pointers and a work-stealing deque that's correct on ARM, with live demos and claims cited to the formal models and vendor manuals.
Time
~75 min
Level
Senior engine programmer
Prereqs
You can read C++ at an intermediate level. You've heard of std::atomic. The Job Systems tutorial uses these primitives; this one teaches what they mean.
Hardware
Some idea that CPUs have caches
◂ Build a Game Engine
Phase 1 · Memory & concurrency
Next · Memory Allocators ▸
What you'll be able to do by the end

    Explain why another thread can see your writes late or out of order (store buffers, caches, the compiler), and which reorderings x86 and ARM each allow.
    Pick the right memory_order for a stats counter, a "data is ready" flag, a reference count and a sleep/wake handshake, and say what each costs on x86 and on ARM.
    Write an atomic reference counter and a single-producer/single-consumer queue that are correct on ARM, with no locks.
    Spot ABA and use-after-free in lock-free code, and fix them with tagged pointers and hazard pointers.
    Read the orderings in a Chase-Lev work-stealing deque and a seqlock and say why each fence is there.


## Sections

- [Why a Memory Model Exists](01-why-a-memory-model-exists.md)
- [A Short History of Getting Cross-Thread Ordering Right](02-a-short-history-of-getting-cross-thread-ordering-right.md)
- [What the CPU Actually Does](03-what-the-cpu-actually-does.md)
- [Sequential Consistency, the Strawman](04-sequential-consistency-the-strawman.md)
- [x86-TSO vs ARM](05-x86-tso-vs-arm.md)
- [The C++ Memory Orderings](06-the-cpp-memory-orderings.md)
- [Acquire and Release: Publish-Subscribe in Two Lines](07-acquire-and-release-publish-subscribe-in-two-lines.md)
- [Sequentially Consistent: The Strong Default](08-sequentially-consistent-the-strong-default.md)
- [The Atomic Reference Counter](09-the-atomic-reference-counter.md)
- [False Sharing and the Cache Line](10-false-sharing-and-the-cache-line.md)
- [SPSC: A Lock-Free Queue with No CAS](11-spsc-a-lock-free-queue-with-no-cas.md)
- [The ABA Problem](12-the-aba-problem.md)
- [compare_exchange_weak vs strong](13-compare-exchange-weak-vs-strong.md)
- [Hazard Pointers: Safe Memory Reclamation](14-hazard-pointers-safe-memory-reclamation.md)
- [The Chase-Lev Work-Stealing Deque](15-the-chase-lev-work-stealing-deque.md)
- [The Seqlock: Optimistic Reads of Compound State](16-the-seqlock-optimistic-reads-of-compound-state.md)
- [Try It Yourself: The Memory-Ordering Playground](17-try-it-yourself-the-memory-ordering-playground.md)
- [How Unreal Does It](18-how-unreal-does-it.md)
- [How the Same Atomic Lowers on x86 and ARM](19-how-the-same-atomic-lowers-on-x86-and-arm.md)
- [Pitfalls](20-pitfalls.md)
- [What's Next](21-whats-next.md)
- [Sources & Further Reading](22-sources-and-further-reading.md)
