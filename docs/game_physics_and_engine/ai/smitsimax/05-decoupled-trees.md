# Decoupled trees

Back to the four pods: instead of one tree with 1,296 children per node, each pod could get a tree of its own with 6. Each pod would pick from its own tree without seeing the others, and the picks would be combined only when the game is simulated.

Smitsimax solves both problems with one structural change: give each player their own search tree. Instead of one tree with interleaved or joint moves, player 1 has a tree where each node represents one of player 1's moves, and player 2 has a separate tree where each node represents one of player 2's moves. In a four-player game, there are four independent trees.

On each iteration:

    Each player independently descends their own tree using UCB1, selecting a move at each depth level.
    The selected moves are combined into a joint action.
    The game state is advanced using that joint action.
    This repeats to max depth (or until terminal).
    The final state is scored.
    Each player's tree is backpropagated independently with that player's score.

The branching factor per tree is just m (the number of actions for that player), not mk. No player sees another's selection before choosing, and every agent's tree gets the same search effort, so the model of the opponent is as strong as the agent's own planning. MSmits lists this symmetry as the method's main advantage[9].

Worked example

One iteration of the widget below. Player 1's tree picks L as its first move and Player 2's tree picks R, each without seeing the other. The joint action (L, R) is simulated: R beats L, so Player 1's score is centered on 2 out of 10, plus up to ±2 of noise, say 2.6. The game is zero-sum, so Player 2 scores 10 − 2.6 = 7.4. Player 1's tree adds 2.6 to every node on its path and Player 2's tree adds 7.4 to every node on its own. Neither tree ever stores the other's move.
Live · Smitsimax dual-tree search
step
play
reset
iteration
0
P1 nodes
4
P2 nodes
4
phase
idle
Click "step" to advance one phase at a time, or "play" to run it, and watch both trees grow side by side. Two independent trees, one per player. Purple highlights show the selected path in each tree. The vertical dashed line in the center is the "wall" between players: neither tree sees the other's selection. After selection, the chosen moves combine (green merge line) and the joint action is scored: the first moves play a Rock-Paper-Scissors-style cycle (L beats M, M beats R, R beats L) plus some noise, and the game is zero-sum, so Player 2 scores 10 minus Player 1's score. Each tree then backpropagates its own player's score (orange flash). Nodes show the average score and visit count; the first 10 visits to a node select at random, as in MSmits's version. As the iterations run, each tree's statistics reflect the other tree's current mix of moves, which is itself adapting.

Each tree "sees" the opponent's strategy only through the statistical distribution of outcomes across iterations. If Player 1's tree discovers that move A leads to high scores, Player 2's tree will eventually discover that its counter to A is best, which in turn shifts Player 1's statistics, and so on. The trees co-adapt through the shared simulation results, without ever directly observing each other's structure.
