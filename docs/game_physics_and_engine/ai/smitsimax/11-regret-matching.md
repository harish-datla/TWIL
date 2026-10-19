# Regret matching

A poker bot that always bluffs in the same spot loses money to anyone who notices, and the same is true of a search that always picks the same move from the same statistics. You want a selection rule that randomizes, and randomizes in the right proportions.

Regret matching is the theoretically grounded alternative to UCB1 for simultaneous-move selection. The idea: after each round, compute the regret for each action, how much better you'd have done by playing it instead of what you played, then select with probability proportional to accumulated positive regret.

The first formula adds up that regret over every round so far. For action a after round t:
Rat = Στ=1..t [u(a, bτ) − u(aτ, bτ)]

In words: for every round so far, take what action a would have paid against the opponent's actual move, subtract what you actually got, and add it all up. Positive means you'd have done better playing a.

The second formula turns those totals into a mixed strategy. The selection probability for action a on round t+1:
σat+1 = [Rat]+ / Σa' [Ra't]+

In words: ignore actions with negative total regret, and play each other action in proportion to how much you regret not having played it.

Worked example

Round 1 of Rock-Paper-Scissors: you throw Rock, the opponent throws Paper, and you get −1. Rock's regret is −1 − (−1) = 0. Paper would have tied for 0, so its regret is 0 − (−1) = 1. Scissors would have won 1, so its regret is 1 − (−1) = 2. The positive regrets sum to 3, so round 2 plays Rock 0%, Paper 1/3 and Scissors 2/3: more weight on the throw that beat what the opponent just did.

If all regrets are zero or negative, play uniformly at random. The convergence theorem (Hart and Mas-Colell, 2000[13]): if both players in a two-player zero-sum game use regret matching, their average strategies converge to a Nash equilibrium.

This is the foundation of Counterfactual Regret Minimization (CFR)[14], the algorithm that solved heads-up limit Texas Hold'em[15] and underlies most modern poker AI. CFR extends regret matching to extensive-form (sequential) games by computing counterfactual regrets at each decision point. A Smitsimax-style search can use regret matching instead of UCB1 at each tree node and inherit the convergence guarantee of §10 (for the average strategy, with enough exploration); whether that wins games within a fixed time budget depends on the game.

CodinGame implementations called Smitsimax keep UCB1, as MSmits's original does. The convergence guarantee matters when you need a strategy that can't be exploited (poker, where any leak costs real money) and matters less in time-limited game AI, where a strong move within 50 ms beats a guarantee about the limit.
