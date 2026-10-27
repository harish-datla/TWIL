# Pitfalls

Common bugs in Smitsimax implementations:

    Forgetting to re-simulate from root each iteration. Because nodes don't map to unique game states, you can't cache the state at a node and pick up from there. Every iteration must clone the initial state and re-simulate the full path of joint actions from depth 0. Caching the state at a Smitsimax node gives you the state from a previous iteration with a different opponent path, producing garbage.
    Updating the wrong tree's statistics. When backpropagating, each tree receives its own agent's score. If you accidentally give Player 2's score to Player 1's tree, the search converges on the opponent's goal instead of your own.
    UCB1 with zero visits. A child node with zero visits produces a division-by-zero in the exploration term. Ensure every child is visited at least once before applying UCB1, or handle the zero-visit case explicitly (return +∞ to force exploration of unvisited children).
    Not normalizing scores. If your game's scores range from 0 to 10,000, the exploitation term dominates UCB1 at any reasonable C value, and the search degenerates into greedy exploitation. Normalize by (Smax − Smin) or use a fixed known range.
    Re-seeding the rollout generator every iteration. If you use random rollouts and seed the generator with the same value at the start of each iteration, every rollout below a given path plays out identically, so the averages measure one continuation instead of many. Seed once per search and let the generator run.
    Choosing the expansion point badly. Expanding a node on its first visit means UCB1 would immediately face fresh children with no data; expanding much later wastes iterations replaying the same path. MSmits expands on the first visit but selects at random until the node has 10 visits[9].
    Allied agents fighting each other. In team games, allies' evaluations should share the team's goals. If each agent maximizes an unrelated objective, allies' trees converge on conflicting strategies. Rewarding each agent partly for its teammates' progress, as MSmits's blocker is rewarded for the runner's, lets cooperation emerge.

