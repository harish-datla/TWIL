# Lock-Free Queues & Memory Ordering

Your job system's workers all pull from one shared queue, and the audio callback has to receive play commands from the game thread without ever taking a lock. Both want a queue that never blocks: the job pool so workers don't wait their turn on a mutex, the audio callback because a lock can make it miss its deadline and the sound glitches. Lock-free queues deliver that, and they are also where concurrency bugs go to hide. This page builds the single-producer, single-consumer ring that needs no compare-and-swap, walks through Vyukov's multi-producer queue, and shows the two traps (ABA, and weak memory ordering) that make hand-rolled lock-free code dangerous, so you can pick the right queue, or the right library.
Time
~35 min
Level
Senior
Prereqs
The Memory Model (atomics, acquire/release) and Job Systems (the thread pool that needs this).
Stack
C++ & Rust
◂ Build a Game Engine
Phase 1 · Memory & concurrency
Next · 3D Math for Games ▸
What you'll be able to do by the end

    Say what obstruction-free, lock-free and wait-free each guarantee, and why a CAS retry loop is lock-free but not wait-free.
    Write a single-producer, single-consumer ring that uses one acquire/release pair per direction and no CAS.
    Explain how Vyukov's MPMC queue uses per-cell sequence numbers, and why it isn't strictly lock-free.
    Recognize the ABA problem and choose between tagged pointers, hazard pointers and epochs.
    Explain why ordering bugs hide on x86 PCs and consoles and show up on Arm, and know when to ship a library instead.


## Sections

- [Why Lock-Free](01-why-lock-free.md)
- [The Memory Model in One Screen](02-the-memory-model-in-one-screen.md)
- [CAS and the Progress Taxonomy](03-cas-and-the-progress-taxonomy.md)
- [SPSC: Reach for This First](04-spsc-reach-for-this-first.md)
- [Bounded MPMC: Vyukov](05-bounded-mpmc-vyukov.md)
- [The ABA Problem](06-the-aba-problem.md)
- [x86 vs ARM: The Latent Bug](07-x86-vs-arm-the-latent-bug.md)
- [When Not to Go Lock-Free](08-when-not-to-go-lock-free.md)
- [Pitfalls](09-pitfalls.md)
- [What's Next](10-whats-next.md)
