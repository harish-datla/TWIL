# BT vs GOAP vs HTN

Sometimes the designer doesn't want to author each decision. They want to say "kill the player" and have the soldier work out "pick up a gun, reload, flank" on its own, in whatever order the level allows. That's what a planner does, and two planning-based alternatives to behavior trees ship in production.

GOAP (Goal-Oriented Action Planning) gives every action preconditions and effects, such as "Reload needs ammo in the bag and leaves the gun loaded", and lets a search find a chain of actions that reaches the goal. HTN (Hierarchical Task Network) planning has designers write how each big task breaks into smaller ones, and a planner picks the breakdown that fits the current world state.
Property	Behavior Tree	GOAP	HTN
Behavior authored by 	Designer builds tree structure 	Designer defines actions + preconditions; planner finds the sequence 	Designer defines task decompositions; planner picks decomposition path
Emergent behavior 	Low. The tree encodes the decision space explicitly. 	High. The planner can combine actions in ways the designer didn't anticipate. 	Medium. Constrained by decomposition hierarchy but can still surprise.
Debuggability 	High. One path through a tree. Visual debuggers show the active branch. 	Low. Plans are generated at runtime. Hard to explain why the planner chose a specific action sequence. 	Medium. The decomposition tree is readable, but plan selection can be opaque.
Shipped in 	Halo 2[1], Hitman (2016)[13], games built on Unreal's BT 	F.E.A.R.[18] 	Transformers: Fall of Cybertron[19]
CPU cost per tick 	Low. One tree walk: about the depth of the active path for memory BTs, up to every node for reactive ones. 	Variable. Planning is a search, potentially expensive. Usually amortized by caching plans. 	Low to medium. The method hierarchy prunes the search; High Moon measured its HTN planner as considerably faster than its earlier GOAP one[19].

F.E.A.R. (Monolith, 2005) is the best-known shipped GOAP game. Jeff Orkin's GDC 2006 talk[18] describes a system where the FSM has only three states (Goto, Animate, UseSmartObject) and A* plans over the action space to decide what to do. The soldiers flank, lay suppression fire and flush the player out with grenades; the cost lands on debugging, where "why did the AI throw a grenade?" means reconstructing the planner's search.

HTN planning (Humphreys, Game AI Pro[19]) sits between BTs and GOAP. High-level tasks decompose into subtasks according to designer-authored rules. The planner picks decomposition paths based on world state. High Moon Studios shipped HTN in Transformers: Fall of Cybertron and reported it "considerably faster" than the GOAP system in the previous game, War for Cybertron, because the task hierarchy culls large parts of the search and the planner needs no heuristic sort.

Guerrilla's Horizon Zero Dawn uses a different hybrid for its machines: HTN planning combined with utility-based decisions, plus a group layer that coordinates herds of different machine classes (§14)[16].
