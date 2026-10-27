# Standard MCTS

Say your bot plays a turn-based game, has three legal moves, and gets 50 ms to pick one. It can't play every line to the end, so it samples: play lots of quick random games from each move, and spend more of the budget on the moves that keep winning. That's MCTS, and it's the baseline SMITSIMAX changes.

Standard MCTS builds a search tree incrementally over many iterations. Each iteration has four phases:

    Selection. Starting from the root, follow the tree down using a selection policy (UCB1, typically) to pick which child to visit at each internal node. Continue until you reach a node that has unexpanded children.
    Expansion. Add one (or more) child nodes to the tree, representing moves not yet explored.
    Simulation (rollout). From the new node, play random moves for both sides until the game ends. Record the outcome.
    Backpropagation. Walk back up from the new node to the root, updating every node's visit count and win/score tally.

After the time budget runs out, the root's most-visited child is usually the chosen move. The widget below shows this loop building a tree for a simple two-player game. Each node shows wins/visits from the point of view of the player who made that move; edges are moves.

Worked example

The widget's game has three moves per turn, and its root starts with its three children: 4 nodes. Each iteration selects a path down to a node with no children, expands it by adding its 3 children, simulates from one of them, and backpropagates. So the node count goes 4, 7, 10, 13 over the first three iterations. If a simulation ends in a Player 1 win, every node on the path gets +1 visit, and the root and the nodes for Player 1's moves also get +1 win, so each level's win rate is the mover's own.
Live · Standard MCTS tree growth
step
play
reset
iteration
0
nodes
4
phase
idle
Click "step" to advance one phase at a time, or "play" to watch it run. This is standard MCTS for a sequential two-player game with branching factor 3. Selection follows UCB1 (purple path); because wins are counted for the player who moved, UCB1 at each level picks the move that is best for the player choosing it. Expansion adds the leaf's children and picks one to simulate from (green). Simulation plays random moves to a terminal state (dashed line). Backpropagation updates every node on the path (orange flash). Rollout results here are weighted coin flips (55% wins for the first player), so the tree grows asymmetrically only because UCB1 chases whichever branches happened to win early; in a real game the same mechanism concentrates search on the strong moves.

The key property: each node in the tree corresponds to one game state. Player 1 moves, then Player 2 responds, then Player 1 again. The tree interleaves moves. This works for chess, Go, and any game where players alternate turns.
