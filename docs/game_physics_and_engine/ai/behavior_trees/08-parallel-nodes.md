# Parallel nodes

A guard sprinting to cover should keep its gun pointed at you while it runs. A sequence can't do that: it would finish the run and only then start tracking you. You need a node that keeps both children going at once.

A Parallel node ticks all its children in the same tick rather than one after another, so several of them can be RUNNING at once. It then needs a rule for when it's done. Two success policies are common, and several implementations generalize both into a threshold (succeed once k children have):

    Succeed-on-all. Returns SUCCESS only when every child has returned SUCCESS: the "all must pass" gate.
    Succeed-on-one. Returns SUCCESS as soon as any child succeeds: the "any will do" gate.

Whether a parallel re-ticks a child that has already returned is an implementation choice rather than a property of the idea, and two widely used open-source runtimes disagree. BehaviorTree.CPP, a C++ library, keeps a set of completed children and skips them until the node resets[20]. py_trees, a Python library, does the opposite by default: "A parallel ticks every child every time the parallel is itself ticked", with skipping available as an opt-in synchronised setting on its SuccessOnAll and SuccessOnSelected policies[21]. Read your runtime before assuming either.

Failure rules vary more than the success policies. py_trees fails the parallel as soon as any child fails, under every policy[21]; BehaviorTree.CPP makes the failure count a second threshold (below). A typical use: [Parallel: MoveTo(cover), TrackTarget]. The agent moves and keeps its aim on the enemy at the same time, and under succeed-on-one, MoveTo finishing resolves the node as SUCCESS and the still-running TrackTarget is halted: stopped, with whatever progress it had thrown away.

BehaviorTree.CPP does not expose named policies at all. It exposes two integer ports, success_count (default -1, meaning all children) and failure_count (default 1), and states the completion rule in one line: "It is completed when either the SUCCESS or FAILURE threshold is reached. Any remaining running children are halted."[20] The source carries a second failure route the docs leave out: the node also fails the moment enough children have failed that the success threshold can no longer be reached[20]. And for the case where the halt is the part you do not want, there is a separate node: ParallelAll "always executes ALL children to completion. It never halts children early."[20]

Worked example

The sandbox below starts with four children: MoveTo succeeds after 3 ticks, PlayAnim after 5, TrackTarget after 2, and Reload fails after 6. With BehaviorTree.CPP's defaults (all four must succeed, one failure fails the node), successes arrive at ticks 2, 3 and 5, and at tick 6 Reload fails: the node returns FAILURE with nothing left to halt. Set success_count to 1 and TrackTarget's success at tick 2 ends it instead. MoveTo (2 of 3 ticks done), PlayAnim (2 of 5) and Reload (1 of 6, not ticked on tick 2 because it sits to the right of the deciding child) are all halted: 3 actions and 5 ticks of their work thrown away.
Live · Parallel policy sandbox
run
step
reset
success_count
1
2
3
4 (all)
failure_count
1
2
3
4
Children: completion tick and outcome
MoveTo
−
3
+
SUCCESS
PlayAnim
−
5
+
SUCCESS
TrackTarget
−
2
+
SUCCESS
Reload
−
6
+
FAILURE
tick
0
resolved
0 ok / 0 fail
parallel returns
RUNNING
children halted
0
Each row is one child and each column one tick. Press run or step, then change the two thresholds or a child's duration and outcome and run it again. Each child has its own completion tick and outcome, and the node follows BehaviorTree.CPP's loop: it ticks the unresolved children in order, skips any child that has already returned (its row goes dim, where py_trees' default would tick it again), and tests both thresholds after every child. The moment one is met, anything still running is halted and loses its progress, marked x, including children to the right that were not ticked that tick. The defaults are BehaviorTree.CPP's: success_count = 4 (all children) and failure_count = 1 (any single failure). With them the node runs until tick 6 and fails when Reload fails. Set success_count to 1 and TrackTarget decides it at tick 2 instead, halting the other three mid-flight: same children and durations, a third of the time on the agent, and three actions thrown away.
Check Your Understanding Q1 of 2

A Parallel node has four children with success_count = 4 and failure_count = 1. Three children are still RUNNING when the fourth returns FAILURE. What happens to the three?
They are halted immediately and lose whatever progress they had accumulated.
They keep running until they resolve, because the parallel cannot report until all four have.
They are ticked once more so they can finish cleanly, then the node returns FAILURE.
0/2 correct
Parallel does not mean multi-threaded

"Parallel" describes the evaluation policy: several children can be RUNNING at once, but they are ticked one after another on one thread, so there is no race between them. BehaviorTree.CPP's documentation says its parallel nodes run children "concurrently, but not in separate threads"[20], and py_trees calls the parallelism "merely conceptual"[21]. Real concurrency, where it exists, lives inside a leaf that hands its work to another thread or process. Unreal avoids the general node: its Simple Parallel pairs one main task with a background subtree[6].
