# Selectors (fallbacks)

A guard has three things it could be doing: attacking, chasing or patrolling. It should attack if it can, chase if it can't attack, and patrol only when neither applies. That "try these in priority order, stop at the first one that works" pattern is a selector.

A Selector (sometimes called Fallback or Priority) is OR logic. It executes children left to right. If a child returns FAILURE, the selector moves to the next child. If a child returns SUCCESS, the selector stops and returns SUCCESS. If all children fail, the selector returns FAILURE. If any child returns RUNNING, the selector returns RUNNING.

The mental model: "try A. If that fails, try B. If that fails, try C." The leftmost child has the highest priority. A top-level selector with [AttackSequence, ChaseSequence, PatrolAction] tries combat first, pursuit second, and only patrols if neither applies.

Worked example

Give a sequence and a selector the same three children, [HasAmmo?, Aim, Fire], returning SUCCESS, SUCCESS, FAILURE. The sequence ticks all three: HasAmmo? and Aim succeed, Fire fails, so it returns FAILURE. The selector ticks only HasAmmo?, which succeeds, so it returns SUCCESS and never looks at Aim or Fire. Same children, opposite answers, and a different number of nodes ticked.

The widget below puts a sequence and a selector side by side with the same three children. Toggle each child's result to see how the composite outcome differs.
Live · Sequence vs Selector
toggle child results (click to cycle)
HasAmmo?: OK
Aim: OK
Fire: FAIL
Sequence result
FAIL
Selector result
OK
Click a child's button to cycle its result through OK, FAIL and RUNNING; both composites read the same three results, and each panel shows how far its composite got. With the default (OK, OK, FAIL): the sequence stops at Fire and returns FAILURE (AND logic, one failure is enough to fail it). The selector stops at HasAmmo? and returns SUCCESS (OR logic, one success is enough). Click HasAmmo? once to make it FAIL and neither result changes, but both paths do: the sequence now stops at the first child, and the selector moves on and succeeds at Aim. Click again for RUNNING, which stops either composite at that child and propagates up unchanged.
Check Your Understanding Q1 of 2

A sequence has children [A, B, C]. A returns SUCCESS, B returns RUNNING. What does the sequence return?
SUCCESS, because its first child already succeeded.
RUNNING, and it will resume at B on the next tick.
FAILURE, because C was never reached this tick.
0/2 correct
