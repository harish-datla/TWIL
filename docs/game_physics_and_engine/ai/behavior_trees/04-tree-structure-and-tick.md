# Tree structure and tick

A guard is halfway through a walk to its next waypoint when the AI update runs again. The tree can't block the game until the walk finishes, and it can't forget the walk either. So every node answers each tick with one of three words: done, couldn't, or still working.

A behavior tree is a rooted tree. Every node has zero or more children. The tree is evaluated from the root every tick (typically once per frame, though some engines tick AI at a lower rate to save CPU). Each node's tick() function returns one of three statuses:

    SUCCESS: the node did its job.
    FAILURE: the node tried and could not do its job.
    RUNNING: the node is still working. Come back next tick.

Node types fall into three categories:

    Composite nodes (Sequence, Selector, Parallel) have multiple children and control evaluation order.
    Decorator nodes (Inverter, Repeat, Cooldown) have exactly one child and modify its result.
    Leaf nodes have no children: either an action (does work) or a condition (tests the world).

Evaluation is top-down, left-to-right. The root ticks its children according to its composite type, those children tick theirs, and the results propagate back up: one walk through the tree per tick.

Worked example

The widget below has 8 nodes: a root selector, an Attack sequence (EnemyClose? then Fire), a Chase sequence (EnemySeen? then MoveTo), and a Patrol action. With no enemy around, one tick visits 6 of them: Root, Attack, EnemyClose? (fails, so Fire is skipped), Chase, EnemySeen? (fails, so MoveTo is skipped), then Patrol, which succeeds. With an enemy in range it visits 4: Root, Attack, EnemyClose?, Fire. Chase and Patrol are never touched, because the selector stops at the first branch that succeeds.

The widget below shows this evaluation in action. A selector at the root tries three branches: Attack (sequence of condition + action), Chase (sequence of condition + action), and Patrol (single action). Switch the scenario to change which conditions pass, then step through or auto-play to watch the tick propagate.
Live · BT tick visualizer
step
auto
reset
scenario
patrol
chase
attack
step
0
total steps
9
scenario
patrol
Pick a scenario, then press step to light up one node result at a time, or auto to play the tick. Total steps counts every highlight, including each composite's entry and its result: 9 for patrol and chase, 6 for attack. In the "patrol" scenario, both conditions fail. The selector tries Attack (fails at EnemyClose?), then Chase (fails at EnemySeen?), then falls through to Patrol (succeeds). In "chase," EnemySeen passes, so the Chase sequence runs. In "attack," EnemyClose passes, so the selector never reaches Chase or Patrol. Priority is encoded by left-to-right order. Blue (EVAL) marks a composite still working through its children within this tick; it is not a return status.
