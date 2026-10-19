# Why behavior trees

An NPC decides, many times a second, whether to patrol a route, chase a target, take cover, reload, call for backup or retreat. In the early 2000s those decisions commonly lived in finite state machines (FSMs): the NPC is in exactly one state, such as Patrol, and a hand-written transition moves it to another, such as Chase. That works at five states. Every state added after that has to be checked against every existing one for transitions in both directions (§3), and by a few dozen states the transition set is hard to review and easy to break.

A behavior tree drops the transitions. Each behavior is a branch, the order of the branches sets their priority, and the game walks the tree from the root on every tick (one AI update, often once per frame) to find the first branch that can run. The survey literature puts it structurally: in a BT "the state transition logic is not dispersed across the individual states, but organized in a hierarchical tree structure, with the states as leaves"[5]. Adding a behavior means grafting a subtree onto an existing branch, not wiring transitions to every other state. (Walking from the root every tick is the classic model; §11 covers the event-driven variants that keep running the active branch and re-check only when something they watch changes.)

You'll meet them in the major engines. Unreal ships a behavior tree editor[6] and, since 2022, StateTree, a hybrid of trees and state machines, alongside it[7]. Unity shipped its own first-party tool in September 2024, the com.unity.behavior package[8], which Unity calls a behavior graph rather than a tree, since branches may merge back. Studios that hand-roll their AI often land somewhere else entirely, as §14 shows.

What the tree buys you over an FSM:

    Modularity. A subtree is self-contained. You can move it, copy it, disable it, or hand it to another designer. FSM states are tangled by their transitions.
    Scalability. Adding a branch to a BT is one edit in one place, however big the tree is. Adding a state to an FSM can mean a transition to and from each existing state.
    Debuggability. A BT's evaluation path is a single top-down walk. You can visualize it as a highlighted path through the tree. An FSM's history is a sequence of state transitions, and the current behavior depends on which transition fired last.
    Re-evaluation. A reactive BT re-checks higher-priority branches every tick. An FSM leaves its current state only through a transition someone authored from that state, so reacting to a new situation depends on that transition existing.

