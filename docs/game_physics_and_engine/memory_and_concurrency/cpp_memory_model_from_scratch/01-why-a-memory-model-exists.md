# Why a Memory Model Exists


A loading thread decodes a texture into a buffer, then sets textureReady = true. The render thread spins until it sees the flag, then uploads the buffer. On your x86 dev box it has never failed. On an ARM device the render thread now and then uploads a half-decoded texture. Neither CPU is broken: the render thread saw the two writes in a different order from the one the loading thread made them in, and nothing in the code told the compiler or the hardware that the order mattered. This page is about how you tell them.

A single-threaded program is easy to reason about. The compiler may reorder instructions for speed, the CPU may execute them out of order, and the cache may hold a write for a thousand cycles before flushing it to memory; none of that is visible. The thread only ever sees the values it last wrote, however aggressively the layers underneath shuffle the execution, and the hardware spends real transistors to keep it that way: store-to-load forwarding hands a load the value of the thread's own not-yet-written store, and memory-ordering checks catch speculative loads that ran too early.

A multithreaded program loses that guarantee the moment a second thread looks at the same memory. The compiler's reorderings now affect what another thread sees, and so does the CPU's out-of-order execution. The store buffer that let the write retire early is now a delay another thread can observe. A memory model is the contract the platform offers about what reorderings are visible across threads, so a programmer can write code that's correct without naming the hardware.

The page works from the hardware up. §3 to §5 show what a CPU does with your loads and stores and which reorderings x86 and ARM let other cores see. §6 to §8 cover the C++ orderings that forbid them and what each one costs. §9 to §16 apply them to the primitives engines are built from: a reference counter, a lock-free ring, ABA and hazard pointers, the Chase-Lev work-stealing deque (a per-worker job queue that idle workers can take jobs from) with the weak-memory orderings from Lê, Pop, Cohen, and Zappa Nardelli (2013)[1], and a seqlock. §17 is a playground where you flip orderings on a litmus test and watch the invariant break.
The textbook example that breaks under threading

The smallest program that shows the problem has two threads and two shared integers initialized to zero. Thread A stores 1 to x then loads y. Thread B stores 1 to y then loads x. At least one of the loads must return 1, right? Both stores happen before any load on the other thread, so when the loads run, at least one of the stores must be visible:
store_buffer_demo.cpp · the intuition that doesn't survive contact

// Shared globals, initialized to zero.
int x = 0;
int y = 0;

// Thread A:
x = 1;             // store
int a_view_of_y = y; // load

// Thread B (running in parallel):
y = 1;             // store
int b_view_of_x = x; // load

// Question: can BOTH a_view_of_y == 0 AND b_view_of_x == 0?
// Intuition says no. One of the stores has to land before the loads run.

On an x86 CPU both loads can return zero. x86-TSO, the formal model of how x86 orders memory, lists this exact test as an allowed outcome[2]. The mechanism is the store buffer, a small per-core queue of writes waiting to reach the cache: when thread A writes x = 1, the store sits in thread A's store buffer for some cycles before it's committed to the cache. Thread A's later load of y can run before that buffered store is visible to thread B, and thread B does the same with its store of y. Both loads finish before either store reaches the cache, so both return zero: an outcome no interleaving of the four operations can produce, with no compiler reordering involved.

Worked example

Times are in the widget's units below, where one store-buffer cycle is 0.05. Thread A stores x = 1 at 0.40 and loads y at 0.42. Thread B stores y = 1 at 0.41 and loads x at 0.43. Both stores issue before either load.

A's store waits one cycle in its buffer and becomes visible at 0.40 + 0.05 = 0.45. B's waits two cycles and becomes visible at 0.41 + 0.10 = 0.51.

A loads y at 0.42, before 0.51, so it reads 0. B loads x at 0.43, before 0.45, so it reads 0. The result is (0, 0), which no one-at-a-time sequence of the four operations can produce (§4 lists all six).

The snippet uses plain ints to keep it short. In C++ that's a data race, which is undefined behavior; with std::atomic<int> the same (0, 0) outcome is allowed for every ordering weaker than seq_cst.

The widget below runs that pattern thousands of times as a model: two threads, each issuing a store and then, a moment later, a load, with a store buffer that can delay the store by up to depth cycles. With depth zero, stores are visible the instant they issue and (0, 0) never appears. Once a store can outlast the gap before its own thread's load, (0, 0) starts showing up:
Live · Store buffer breaks intuition
store buffer depth 2
trials per click 2000
RUN ▶
Reset ⟲
(1, 1) both saw store
0 (0.00%)
(0, 1) or (1, 0)
0 (0.00%)
(0, 0) both saw stale
0 (0.00%)
The two rows are one trial's timelines: the shaded bar is the time a store spends in the buffer, and the strip at the bottom is the share of each outcome over every trial. Set the depth to 0 and press run: (0, 0) never appears. At the default depth of 2 it shows up in about 6% of trials, and at depth 8 in about 31%. A simulator, not a real CPU trace. Each thread issues its store at a random time and its load a fixed short gap later; the store stays in the store buffer for a random 0 to depth cycles before the other thread can see it, and loads read coherent memory. With depth 0 you get the sequentially consistent answer: (1, 1) when the two threads overlap, a mixed result otherwise, never (0, 0). With depth ≥ 1, (0, 0) appears whenever both stores are still buffered when the loads run, and its rate grows with depth. x86-TSO allows the outcome[2]; the rates here come from the model, not from hardware. The timeline shows a (0, 0) trial whenever the run produced one.
Check Your Understanding Q1 of 2

Two threads run the Dekker-pattern code in the snippet above on a stock x86 CPU. After many runs, you see the outcome where both loads returned zero. Which is the most accurate diagnosis?
The compiler reordered the store and the load. Use volatile to fix it.
Each core’s store buffer held the store for a few cycles before the cache saw it, and each core’s load completed before the other core’s store became visible. This is x86-TSO behavior; the fix is a std::atomic_thread_fence(memory_order_seq_cst) (MFENCE) between the store and the load on both threads, or seq_cst atomics throughout.
There is a bug in the CPU. Modern hardware shouldn’t allow this.
0/2 correct

[Back to The C++ Memory Model from Scratch](index.md)
