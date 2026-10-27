# Dependencies as a DAG


Throw a frame's jobs at the workers in any order and draw-list building starts before culling has decided what's visible. Real frame work has structure. Visibility culling has to finish before draw-call building can start. Animation has to finish before skinning. Audio listener position depends on the player's transform, which is updated by gameplay. The graph of which job depends on which is a directed acyclic graph (DAG), and a real scheduler has to honor it.

A sketch of a typical frame's job graph (the same graph as the "typical frame" preset in the demo further down):
input gameplay AI physics animation culling audio draw lists shadows present

The graph fans out early. A fixed-thread design runs each box on its assigned thread, so ready boxes that share a thread wait for each other; a job system can run every ready box on any free core.
Three ways to express dependencies

Different engines pick different syntax for the same idea. The mechanics map onto a small handful of primitives:

    Counters (Naughty Dog). Each job decrements a shared atomic counter when it finishes. A waiting job parks its fiber until the counter reaches the value it waits for, usually zero[2]. The counter is the dependency primitive, and waiting is explicit in the job's code.
    Handles (Unity). Every Schedule() call returns a JobHandle. You pass it into the next Schedule() call as a dependency, or combine several handles with JobHandle.CombineDependencies(a, b, c). The scheduler tracks the resulting DAG and runs each job once its predecessors complete[8].
    Continuations (Unreal Tasks, C++26 senders). Each task carries a list of prerequisite tasks. When a prerequisite completes, it decrements a counter on its dependents; whichever decrements to zero gets enqueued. Expressed this way, no worker blocks waiting for a prerequisite[9].

Handles and continuations come down to the same bookkeeping: each task counts its unresolved prerequisites, each finishing prerequisite decrements its dependents' counts, and whichever count reaches zero is enqueued. Naughty Dog's counters invert it: the waiter parks on a count of outstanding jobs, which is affordable because a parked fiber doesn't tie up a worker (§9).

In code, the heart of a dependency-tracking scheduler is about a dozen lines:
deps.cpp · core of a DAG scheduler

// A Task is a node in the dependency DAG. Three fields:
//   1. The callable to run (its "body").
//   2. How many holds still keep it from running.
//   3. Which other tasks are waiting on this one.
struct Task {
  std::function<void()>   body;
  std::atomic<int>        openHolds;    // unfinished prereqs + 1 (the builder's)
  std::vector<Task*>      dependents;   // who depends on us
};

// Drops one hold. fetch_sub returns the value BEFORE the decrement, so a
// return of 1 means this call took the count to 0. Exactly one caller (the
// builder or the last prereq to finish) sees that, and only it enqueues.
void releaseHold(Task* task) {
  if (task->openHolds.fetch_sub(1, std::memory_order_acq_rel) == 1)
    pushToReadyQueue(task);             // nothing left to wait for: run it
}

// Contract: the builder creates every task with openHolds = (its prereq
// count) + 1 and fills in every dependents list first, then calls this once
// per task. The extra hold stops a fast prereq from releasing a task twice.
void submitWhenReady(Task* task) {
  releaseHold(task);                    // drop the builder's hold
}

// A worker calls this when it pops a task from a queue.
void runTask(Task* task) {
  task->body();                         // the actual work
  // acq_rel: the release half publishes this body's writes, and the acquire
  // half lets the last decrementer see every prereq's writes.
  for (Task* dependent : task->dependents)
    releaseHold(dependent);
}

Three fields per task and three short functions. The extra builder hold matters: without it, a prerequisite that finishes while the graph is still being wired can release a dependent early, or release it twice. §10 adds the worker pool, the work-stealing deques, and a small lock per task so a task can be linked to a prerequisite that is already running.

Build a small graph and run it through the scheduler:
Live · DAG Scheduling
graph
workers 4
Run frame ▶
Reset ⟲
elapsed
0.0 ms
critical path
8.6 ms
workers used
0
Boxes light up green when ready, orange while running and gray when done. Pick the typical frame and press Run at 1, 2 and 4 workers, comparing elapsed against the critical path: past 3 workers the frame stops getting shorter. The elapsed clock advances in animation steps, so it lands up to about a millisecond above the worked example's exact numbers. The critical path is the longest chain of dependent jobs. No matter how many workers you have, the frame can't finish in less than the critical-path length. More workers help only up to that floor.
Work and span

Two numbers describe a parallel computation. Work (T1) is the total time on one core. Span (T∞) is the time on infinite cores, which equals the critical-path length. No schedule on P cores can beat the larger of T1/P and T∞; a greedy scheduler finishes within T1/P + T∞, and randomized work stealing achieves T1/P + O(T∞) in expectation[20]. The first term is "we have to do the work"; the second is "we can't beat the longest dependency chain." Engine performance work is about shrinking either of those.

Worked example

The widget's typical frame has ten jobs adding up to 17.0 ms of work. Its longest chain is input 0.4 + physics 3.0 + culling 2.5 + draw lists 2.4 + present 0.3 = 8.6 ms, the widget's critical-path readout.

On 4 workers the floor is the larger of 17.0 / 4 = 4.25 ms and 8.6 ms: 8.6 ms, and a greedy schedule reaches it exactly. On 2 workers the floor is max(8.5, 8.6) = 8.6 ms, but the schedule takes 9.8 ms: the greedy scheduler starts gameplay and AI first, and physics, the head of the critical path, waits 1.2 ms for a free worker. A scheduler that knows the graph can start critical-path jobs first to avoid this. On 1 worker it's the full 17.0 ms.

Going from 4 workers to 8 changes nothing. To speed this frame up, shorten the chain, for example by splitting physics; more cores won't help.

[Back to Job Systems from Scratch](index.md)
