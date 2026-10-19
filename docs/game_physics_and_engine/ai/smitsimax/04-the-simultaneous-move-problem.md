# The simultaneous-move problem

Now the two players pick at the same time, and neither sees the other's choice until both are locked in. The tree above assumes someone moves first, so that assumption breaks. Consider a game where Player 1 picks from {A, B, C} and Player 2 picks from {X, Y, Z}, simultaneously. The outcome depends on the joint action (A,X), (A,Y), ..., (C,Z), nine combinations total.

There are two direct ways to force this into a standard MCTS tree, and both have a serious flaw:
Approach 1: Sequential UCT (SUCT)

Serialize the simultaneous game into a sequential one. Player 1 picks first, then Player 2 responds. This creates a valid tree, but Player 2 now has an information advantage that doesn't exist in the real game. In Rock-Paper-Scissors under SUCT, the tree lets Player 2 see Player 1's throw before choosing, so inside the search Player 2 always wins and every Player 1 throw looks equally lost. The algorithm solves a different game from the one being played.
Approach 2: Joint-action tree

Each node has one child per joint action. For 3 actions per player, that's 9 children. For 10 actions per player, 100 children. For k players with m actions each, mk children per node. The branching factor explodes exponentially in the number of players. A four-player game with 20 actions each has 160,000 children per node. The tree is too wide to explore meaningfully in any reasonable time budget.

Worked example

Coders Strike Back has four pods, two per player. Say each pod's continuous controls are discretized to 6 moves. A joint-action tree then has 64 = 1,296 children per node, and three turns deep it has 1,2963 ≈ 2.2 billion leaves. Four separate trees, one per pod, have 6 children per node and 63 = 216 leaves each, 864 in all.
Live · Joint-action branching factor
actions per player 10
2 players
100
3 players
1,000
4 players
10,000
Drag the slider to change how many actions each player has; the bars and readouts show the joint-action tree's children per node for 2, 3 and 4 players (100, 1,000 and 10,000 at the default 10 actions). The branching factor of a joint-action tree is mk. Even modest action counts become intractable for more than two players. Smitsimax sidesteps this by keeping each player's tree independent: the per-player branching factor stays at m.
Check Your Understanding Q1 of 1

In a 3-player game where each player has 15 possible actions, what is the branching factor of a joint-action MCTS tree?
45 (15 × 3)
3,375 (15³)
225 (15²)
0/1 correct
