# What's next

Next in the series, Navmesh & Steering turns decisions into movement: where the walkable surface comes from, how a path becomes a smooth walk, and how agents move without bumping into each other. Beyond that, simultaneous-move search itself goes further:

    ISMCTS for hidden information. If your game has fog of war, hidden cards, or private knowledge, combine Smitsimax's decoupled trees with ISMCTS's determinization[4]: sample a consistent world state at the start of each iteration, then run Smitsimax on that sample.
    Neural network evaluation. Replace the random rollout with a learned value function, as in AlphaZero. Becker and Sunberg's Simultaneous AlphaZero[11] does this for two-player zero-sum games, pairing the value function with a regret-matching solver for the matrix game at each node.
    CFR integration. For games where Nash equilibrium convergence matters (poker, auctions), use CFR[14]-style regret updates instead of UCB1. Lanctot et al.'s Online Outcome Sampling (OOS), an online search form of Monte Carlo CFR, provably converges to a Nash equilibrium in simultaneous-move games[7].
    Equilibrium approximation at scale. Yu et al.'s 2024 work[10] combines neural-network-guided search with no-regret learning, approximating a coarse correlated equilibrium inside the search. It is one recent attempt to combine theoretical grounding with performance on large benchmarks such as Google Research Football and StarCraft.
    Multi-objective Smitsimax. Some games have multiple objectives (score, survival, territory). Multi-objective MCTS techniques apply to Smitsimax: maintain several scores per node and select on a weighted combination or on Pareto dominance.

For the game AI programmer, the most useful next step is to implement Smitsimax on a simple simultaneous-move game (Goofspiel, or a grid-based chase game) and tune it. The core loop is short (the pseudocode in §7 is about 40 lines); the work is in a fast simulator, the game-specific evaluation function, and tuning C, depth, and the rollout policy. 
