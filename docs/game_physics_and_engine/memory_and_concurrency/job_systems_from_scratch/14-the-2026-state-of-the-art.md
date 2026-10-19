# The 2026 State of the Art


A scheduler you write today will outlive at least one more C++ standard, so it's worth knowing where the vocabulary is heading. Three developments are worth following.
C++26 senders and receivers

In June 2024 at the WG21 St. Louis plenary, the C++ standard committee adopted std::execution (paper P2300) into the C++26 working draft[10], and a year later added a shared parallel scheduler (P2079)[44]. Together they give the standard library a composable model for asynchronous work and the execution resources it runs on. Briefly:

    A sender is a lazy description of an async operation. Think "a function object for sync code, but for async."
    A receiver is a bundle of callbacks: set_value, set_error, set_stopped.
    A scheduler is a factory for senders that complete on a particular execution context (CPU thread pool, GPU, custom).
    Composition is via algorithms like then, let_value, when_all and bulk; std::this_thread::sync_wait runs a sender and blocks for its result. (start_detached and ensure_started were removed before adoption in favor of async scopes.)

stdexec_sketch.cpp · what a C++26 job graph looks like

namespace ex = std::execution;

// C++26's shared thread pool (P2079). With stdexec today, use
// exec::static_thread_pool and its get_scheduler() instead.
auto scheduler = ex::get_parallel_scheduler();

auto physics   = ex::schedule(scheduler) | ex::then([] { return runPhysics(); });
auto animation = ex::schedule(scheduler) | ex::then([] { return runAnimation(); });
auto render    = ex::when_all(std::move(physics), std::move(animation))
               | ex::then([](auto&& physicsResult, auto&& animationResult) {
                   buildDrawLists(physicsResult, animationResult);
                 });

std::this_thread::sync_wait(std::move(render));   // start the graph, block until done

NVIDIA's stdexec is the reference implementation and you can use it today[29]. The CUDA scheduler in the same repo shows the design's intended payoff: the same sender pipeline can target a CPU thread pool or a GPU by swapping the scheduler. Whether engines adopt it is a separate question, but the vocabulary is now standard, so library authors can write portable async code without shipping their own runtime.
Structured concurrency

Nathaniel Smith's 2018 essay "Notes on structured concurrency"[30] argued that unconstrained spawn (or go, or async) primitives are the concurrency analog of goto: they leak control flow out of lexical scope, making cancellation, error propagation, and resource cleanup intractable. The alternative, first built in Smith's own Trio library for Python, is a "nursery": a scope that owns a set of child tasks and cannot exit until all of them complete. Kotlin coroutines and Swift concurrency adopted the same model, and C++'s senders/receivers design is built around it. For job systems, the practical impact is: an exception thrown by a leaf job propagates cleanly back to its parent, and cancellation of the parent cancels every child. Both are surprisingly hard to get right with raw thread pools.
GPU work graphs

The same idea is now appearing on the GPU. D3D12 work graphs, shown at GDC 2024[31], let a shader enqueue more shader work directly, as a graph of nodes, without a round trip to the CPU; the GPU schedules the nodes itself.
Why this section matters even if you skip the libraries

Even if an engine never adopts std::execution, its vocabulary maps onto the job systems on this page: a sender is a task, a scheduler is a worker pool, when_all is a dependency join, and a structured scope is wait(). Knowing the mapping makes other job systems easier to read.

[Back to Job Systems from Scratch](index.md)
