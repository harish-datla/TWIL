# Tuning

Once the simulator and the evaluation function work, a handful of settings decide how well the loop spends its turn budget, and each one trades search breadth, depth or speed against the others. Five parameters control Smitsimax performance:
1 · Exploration constant (C)

The UCB1 exploration parameter. Too low, the search gets stuck in a local optimum. Too high, it wastes iterations exploring bad moves. There is no universal value: the SM-MCTS comparisons tune it separately for every game and variant[6][7], and the right value depends on how scores are scaled. With the exploitation term normalized to about [0, 1], the theoretical √2 is a reasonable first value to tune from.
2 · Max search depth

How many turns ahead to simulate per iteration. Deeper search sees further but spends more time per iteration, reducing the total number of iterations, and MSmits warns that with too much depth the search can't converge: the deepest levels never get enough visits to leave the random phase[9]. For scale, the SnakeByte winner searched to a maximum depth of 12 moves at about 50,000 simulated turns per 50 ms, while the fifth-place bot let the depth grow over the iterations with no hard limit[21].
3 · Random selection threshold

The number of visits to a node before switching from random to UCB1 selection. MSmits uses 10[9]. Setting it to 0 (pure UCB1 from the start) invites the resonance problem from §6: all the trees commit early to moves that only look good against each other's equally poor early choices.
4 · Score normalization

Normalizing the exploitation term by (Smax − Smin) keeps the UCB1 formula balanced regardless of score scale. Without normalization, the exploration constant C has a different effective strength for different games or different stages of the same game. Tracking the running bounds costs a little per iteration; MSmits later dropped it from his Coders Strike Back bot as "kind of heavy" but recommends starting with it[9]. A fixed, known score range does the same job for free.
5 · Rollout policy

The original Smitsimax description expands all the way to max depth on every iteration (no random rollout). At RoboStac's suggestion, MSmits later switched to expanding one node and rolling out randomly from there, which worked a little better[9]. The tradeoff is simulation fidelity vs iteration count: guided rollouts produce more accurate estimates per iteration but are slower, reducing the total number of iterations in a fixed time budget.
