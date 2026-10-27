# Fibers and Why Naughty Dog Uses Them


A draw-list job gets halfway through and needs the result of a culling job that's still running on another core. So far our scheduler is built on threads alone. There's a problem with that model, and it shows up the first time you write a job that wants to wait for another job to finish in the middle of its own work.

What should the draw-list job do while it waits?

    Block on a condition variable. The worker sits idle waiting for the culling job, even though the queue is full of other work, so its core is wasted.
    Spin-wait. The worker burns CPU (and its SMT sibling's share of the core: the second hardware thread sharing the same physical core) watching a flag, which wastes the core just as thoroughly.
    Run other jobs while waiting. The worker pops and runs other jobs until the counter clears; Molecular Matters' Wait() works this way[7], as does §10's wait(). The core stays busy, but the waiting job's frame stays underneath whatever runs on top of it, so it can't resume until that nested work unwinds, and deep nesting can overflow the stack.
    Use continuation-passing. Restructure the draw-list job into two jobs: the part before the wait, and a continuation that runs when culling finishes. Works, but it's invasive to the caller and you can't suspend in the middle of a complex function.

None of those is what Naughty Dog reached for. They wanted to write code like this:
job_with_wait · the "obvious" code

void buildDrawList() {
  prepareCommandBuffer();
  JobCounter* cullingDone = scheduleCulling();
  waitForCounter(cullingDone, 0);   // reads like a blocking wait; with fibers it isn't
  encodeVisibleObjects();
}

"Wait for the counter" looks blocking, but they wanted it to not waste the worker. Their answer was fibers: lightweight user-mode threads that you switch by hand, in user space, with no kernel involvement.
What is a fiber

A fiber is a stack and a saved register state. Switching from fiber A to fiber B means: save A's CPU registers onto A's stack, load B's registers from B's stack, jump to where B was suspended. The OS doesn't know. With hand-written assembly, the switch is on the order of tens of cycles; Boost.Fiber documents "less than a hundred"[23]. Compare a thread context switch, which crosses the user-kernel boundary and costs roughly a microsecond, more once you count the TLB (the CPU's cache of address translations) and cache penalty on wake-up.

On Windows, fibers are an OS API (ConvertThreadToFiber, CreateFiber, SwitchToFiber), implemented in user mode: the kernel doesn't schedule them[21]. Elsewhere you can use Boost.Context (mostly hand-written assembly per ABI) or write the switch yourself in a few dozen lines of assembly per platform. Marl, Google's open-source job system[22], has implementations for x86, x64, aarch64, mips, ppc, riscv, and more in its osfiber_asm_*.S files.

Worked example

A frame has 1,000 places where a job waits on another.

If each wait costs a thread context switch at roughly 1 µs, that's 1 ms of switching per frame, 6% of a 16.6 ms frame, before counting the cache misses after each wake-up.

If each wait is a fiber switch at under 100 cycles, about 33 ns at 3 GHz, it's 1,000 × 33 ns ≈ 33 µs, about 30 times less.
The Naughty Dog architecture

Putting it all together, Gyrling's GDC 2015 talk describes:

    6 worker threads, pinned one per PS4 core. The threads are the execution units.
    160 fibers total: 128 with 64 KiB stacks for ordinary jobs, 32 with 512 KiB stacks for jobs that need more stack.
    3 priority queues (low, normal, high). No work-stealing in the classical Cilk sense; the queues are global.
    Atomic counters are the synchronization primitive. waitForCounter(n) doesn't block the worker; it parks the current fiber on the counter and switches to another fiber. When the counter drops to n, the parked fiber becomes runnable again[2].

The result is that jobs can wait inline, in normal-looking code, without wasting workers: a worker waiting on a job never blocks on an OS primitive; it switches fibers.

Step through a fiber switch:
Live · Fiber Switching
Step ▸
Run ▶
Reset ⟲
idle. press Step to advance.

One worker, three fibers. The worker switches between them as each fiber waits on a counter. Dotted lines show suspended call stacks.
Each fiber preserves its full call stack across a switch. That is what makes waitForCounter() callable from the middle of any function, no matter how deep in the call graph, without the caller knowing.
Fibers versus C++20 coroutines

A natural question in 2026: why not C++20 coroutines instead of fibers? The two differ in where the suspended state lives.

    Stackful (fibers). Each fiber has a real stack of fixed size. Any function can suspend the fiber, no matter how deep. Switching is fast but the stack is allocated up front[23].
    Stackless (C++20 coroutines). The compiler transforms the coroutine function into a state machine. Only that function can suspend; if you want a callee to suspend, you have to mark it a coroutine too, and so on up the chain. Each coroutine frame holds only the state that lives across a suspension point, and it is heap-allocated unless the compiler can elide the allocation.

For a game engine job system, the choice usually comes down to: do you want to be able to call waitForCounter() from anywhere in a job's call stack? Fibers say yes; coroutines say only from coroutine-marked frames. Naughty Dog's engine, FiberTaskingLib and Marl use fibers for this reason: engines have a lot of code that wasn't written with suspension in mind, and making every frame on the path a coroutine is invasive.
Fibers aren't free

Each fiber stack is real memory. Naughty Dog's 160 fibers cost 128 × 64 KiB + 32 × 512 KiB ≈ 24 MiB just for stacks, before any actual work. The choice is which cost you'd rather pay: memory for preallocated stacks, or marking every function on a suspending path as a coroutine.
Check Your Understanding Q1 of 2

Why does Naughty Dog use fibers instead of just blocking a worker thread on a condition variable?
Fibers are smaller in memory.
If a worker blocks on a condition variable, that core sits idle until the condition is signaled. A fiber-aware waitForCounter() can switch to a runnable fiber instead, so the core stays busy.
Fibers are easier to debug.
0/2 correct

[Back to Job Systems from Scratch](index.md)
