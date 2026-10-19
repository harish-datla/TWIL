# Memory and reactive behavior

A guard is three ticks into a five-tick walk to its next waypoint when you step into view. Does it turn and fight on the next tick, or finish the walk first? That depends on whether the composites above the walk re-check their earlier children every tick, and each choice has a cost.

Two evaluation strategies:

    Reactive. The composite re-ticks its earlier children every tick, so a condition that has changed since last tick is seen immediately. The benefit is responsiveness: the agent reacts within one tick. The cost is that every condition to the left of the running child is re-evaluated every frame.
    With memory. The composite records which child was RUNNING and, on the next tick, skips straight past the children that already completed. The benefit is efficiency and not re-running work. The cost is missed priority changes: if an enemy appears while a patrol action is RUNNING, the combat branch is not reconsidered until the patrol resolves or something aborts it.

Two things this framing gets wrong if you read it too fast

First, these are properties of an individual composite, not modes of the whole tree. The literature and the reference implementations both spell this out as separate node types: BehaviorTree.CPP ships Sequence, ReactiveSequence and SequenceWithMemory as separate nodes that differ in what they re-tick after a child returns FAILURE or RUNNING, plus AsyncSequence, a Sequence that yields back to the tree after each child that succeeds[9]. A real tree mixes them.

Second, a composite with memory still ticks from the root. Every tick enters at the root and walks down through the composites on the active path; what memory changes is that a composite skips children it has already resolved, not that the traversal starts somewhere else. Colledanchise and Ögren make the same point from the other direction, noting that memory nodes "can be considered to be syntactic sugar" because their behavior is reproducible with a memoryless tree plus auxiliary conditions[3].

And the trap that follows from the first point: a reactive composite re-ticks its earlier children, so any action to the left of the running one runs again every frame. Fine for a condition check. Not fine for firing a weapon, spending ammo, or playing a bark. Non-idempotent actions under a reactive parent are a classic BT bug.

Worked example

Take the abort playground's tree below: a Root selector over Combat (EnemySpotted?, then a four-tick Engage) and Patrol (a five-tick GoToWaypoint, then a three-tick Scan that starts on the tick the walk ends). With no enemy, Patrol takes 7 ticks. A reactive root checks EnemySpotted? on all 7 of them. A memory root checks it once, on tick 1, then resumes Patrol for ticks 2 to 7. If the enemy appears just after tick 1, the reactive root starts Combat on tick 2, while the memory root keeps patrolling for 6 more ticks and starts Combat on tick 8. Seven condition checks buy a one-tick reaction; one check costs up to six ticks of delay.

Production systems tend to mix the two. Unreal's BT does not re-evaluate from the root every tick: the active task keeps running until it finishes or a decorator aborts it, which behaves like memory composites throughout. Each decorator carries an Observer Aborts setting, and Epic's node reference defines the four values exactly: None, "Do not abort anything"; Self, "Abort self and any subtrees running under this node"; Lower Priority, "Abort any nodes to the right of this node"; Both, "Abort self, any subtrees running under this node, and any nodes to the right of this node."[22] When the decorator's observed blackboard key changes, the configured abort fires and forces re-evaluation. That buys reactive behavior exactly where you asked for it, without re-checking every condition every frame.[6]

An abort is not free, and the widget's log is where that shows. BehaviorTree.CPP defines an asynchronous action as one that "May return RUNNING instead of SUCCESS or FAILURE, when ticked" and "Can be stopped as fast as possible when the method halt() is invoked", and notes that "Frequently, the method halt() must be implemented by the developer"[23]. When another node triggers a halt, "the onHalted() method is invoked"[23], and whatever the action had accumulated is yours to unwind: a path request in flight, a partially played animation, a reserved cover slot. Aborting a MoveTo three ticks into five throws away three ticks of pathing. A condition that flickers can trigger that every frame, which is one reason abort types are chosen per decorator rather than set once for the whole tree.

Interruption is where BTs are weakest. Anguelov's slide at GDC 2017 says behavior trees "are inherently bad at two things: Transitions/Interruptions, Behavior Prioritization"[11], and his written follow-up gives the mechanism for each[14]. For transitions: event-driven BTs stopped re-evaluating the tree from the root every update, which fixed the cost, but "behavioral switches will not occur until the currently selected branch completes or fails". The usual repair is monitor nodes that poll conditions outside the running branch and reset the tree when one fires, and "the deeper in the tree we are the more monitors we have registered and therefore the greater the evaluation cost of the tree for each update". For prioritization: forcing one behavior ahead of the tree's fixed order typically takes a global monitor that sets override flags every branch must check, which adds implicit transitions between unrelated branches. In his words, "we have implicitly converted the BTs into an expensive, unreadable and error-prone finite state machine". That is the case for StateTree and the other hybrids in §16.
Predict

Before touching the widget: the root is a selector with memory, the decorator's Observer Aborts is None, and the agent is three ticks into a five-tick GoToWaypoint when EnemySpotted becomes true. How many ticks pass before the Combat branch runs? Then check the two ways to make that number 1, and what each one costs.

The widget below is a real evaluator: actions carry progress across ticks and return RUNNING until they finish, composites remember which child was running, and halt() throws that progress away and says so in the log. The Root and Patrol modes are switchable; Combat is a memory sequence, as an Unreal branch gated by a decorator would be.
Live · Abort playground
run
tick
toggle EnemySpotted
reset
Root selector
memory
reactive
Patrol sequence
memory
reactive
Decorator AbortType
None
LowerPriority
Self
Both
tick
0
EnemySpotted
false
running branch
none
reset
The tree shows each action's progress and each composite's mode, and the log underneath records every tick. Press tick three times, then toggle EnemySpotted, then keep ticking (or press run). The default setting is the one to try first. Root selector on memory with AbortType None: flipping EnemySpotted three ticks in changes nothing for four more ticks, until Patrol finishes both its actions, because a memory composite resumes at its running child and never looks left. Switch the root to reactive and the same flip is caught on the next tick: the selector re-ticks Combat, switches branches, and the log shows halt(GoToWaypoint) discarding its progress. Leave the root on memory and set AbortType to LowerPriority instead, and the decorator's observer fires the moment the key changes, which is Unreal's approach: reactivity where it was asked for rather than everywhere. Self and Both matter on the way out: with Engage running, clear EnemySpotted and only those two abort it; under None or LowerPriority the memory Combat sequence finishes Engage against an enemy that is gone. Finally, put Patrol in reactive mode: GoToWaypoint completes, Scan gets one tick, then the reactive sequence re-ticks GoToWaypoint, which starts a fresh walk and halts Scan, so Patrol never finishes. GoToWaypoint is not idempotent, and this is the §11 bug.
Check Your Understanding Q1 of 3

A patrolling agent uses a memory root selector and a blackboard decorator with Observer Aborts set to None. EnemySpotted flips to true while a five-tick GoToWaypoint is running. When does combat start?
Not until Patrol finishes, because a memory composite resumes at its running child.
On the next tick, because the blackboard write itself re-evaluates the tree from the root.
On the next tick, because a selector always re-checks its higher-priority children first.
0/3 correct
