# The FSM problem

A designer asks for one more enemy state, Heal, in an FSM that already has seven: Idle, Patrol, Chase, Attack, Flee, Stunned and Search. Before anyone writes Heal, someone has to answer fourteen questions: can each of the seven existing states move to Heal, and can Heal move back to each of them? A wrong answer either way is a bug that playtesting finds later.

An FSM with n states has up to n(n - 1) directed transitions, because each state can move to each of the other n - 1. Not every pair needs a transition in practice, but every new state raises the same two questions for every existing one. Missing a transition is a bug (the agent gets stuck in a state). Adding a redundant one is also a bug (the agent escapes a state it shouldn't leave). That's n - 1 pairs to check per new state, and the total grows with the square of the state count, O(n2).

Worked example

The seven-state FSM above has up to 7 × 6 = 42 transitions. Adding Heal makes it 8 × 7 = 56, so 14 new transitions to decide: 7 into Heal and 7 out of it. A behavior tree for the same eight behaviors, as a flat selector, has a root plus one leaf per behavior: 9 nodes and 8 edges. Adding a ninth behavior adds 1 node and 1 edge, and no existing branch changes.

Hierarchical FSMs (HFSMs) help by nesting a smaller state machine inside a state, so a transition every child shares, such as "any combat state can flee", is written once on the parent. The explosion still happens within each level. Pushdown automata, state machines with a stack of states, help with "go do X, then come back to what you were doing". Neither removes the core problem: the interaction between states lives in transitions, and transitions grow with the square of the state count.

The widget below makes this visible. Add states to the FSM on the left and the transition arrows multiply. The behavior tree on the right adds a leaf for each new behavior, and its edge count grows by one each time.
Live · FSM vs BT scaling
+ add state
- remove
reset
FSM states
3
FSM transitions
6
BT nodes
4
BT edges
3
Click + add state up to the maximum of eight and watch the two counts. The FSM on the left draws every possible transition, the worst case: 3 states make 6 arrows and 8 make 56. Real FSMs wire fewer, but whether each pair needs one is still a question someone has to answer, which is the design cost from the text. The BT on the right is a flat selector for simplicity, so 8 behaviors is 9 nodes and 8 edges. A real tree has nested composites, but its edge count still grows by one per behavior, and each new branch is isolated from its siblings.
Check Your Understanding Q1 of 2

You add a seventh state to a fully connected FSM. How many new transitions do you need to wire?
6, one outgoing transition from the new state to each existing state.
12, one transition into the new state and one out of it, per existing state.
7, one transition per state in the machine, counting the new one itself.
0/2 correct
