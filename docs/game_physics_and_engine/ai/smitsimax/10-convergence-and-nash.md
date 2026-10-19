# Convergence and Nash

In a contest your bot meets the same opponents for hundreds of games, and a strong opponent will learn its habits. So the question that matters is whether the search's answer approaches a strategy nobody can exploit: does Smitsimax converge to a Nash equilibrium as iterations increase?

Not in general. Shafiei, Sturtevant, and Schaeffer gave a counterexample in 2009[22]: in plain Rock-Paper-Scissors, UCT self-play happens to select each action a third of the time, matching the equilibrium, but in a biased version whose only equilibrium is (1/16, 5/8, 5/16) it settles into a balanced cycle that is not an equilibrium and that a Nash player can exploit. The frequencies it lands on even depend on each player's exploration constant. Two properties of UCB1 are behind this. It is deterministic: given the same statistics it always picks the same child, so an opponent who knows the statistics can predict and counter every choice. And its visit counts are driven by optimism about under-sampled moves, not by a guarantee that tracks the equilibrium mix. Lisý et al. cite this counterexample as the reason to look for other selection rules[5].

The rules with guarantees are Hannan consistent: their average regret against every fixed alternative action shrinks to zero whatever the opponent does. Lisý et al. prove that SM-MCTS with such a rule, plus enough exploration, drives the average strategy to an approximate Nash equilibrium[5]. Kovařík and Lisý later showed that Hannan consistency on the observed payoffs is not enough on its own; averaging over joint actions, or an extra property that the common algorithms have, closes the gap[16]. The two standard choices:

    Exp3[12] (Exponential-weight algorithm for Exploration and Exploitation): selects stochastically, weighting each action by the exponential of its estimated cumulative reward, mixed with a little uniform exploration.
    Regret Matching[13] (Hart and Mas-Colell, 2000): selects with probability proportional to positive cumulative regret. If both players use regret matching in a two-player zero-sum game, their average strategies converge to Nash.

In head-to-head play the picture is mixed. Across nine GGP games, Tak et al. found Decoupled UCT (the Smitsimax structure) the best performer overall despite its lack of guarantees, with rankings that shifted by game and opponent[6]. In Goofspiel, a card game of secret simultaneous bids where mixing matters, regret matching beat it[7]. One plausible reason DUCT does well: convergence guarantees describe behavior as iterations go to infinity, while under a budget of 50 ms to 1 second UCB1's habit of piling visits onto promising branches pays off. The gap matters most when an opponent adapts to your strategy over many rounds.
Live · Convergence race: UCB1 vs Exp3 vs Regret Matching
start
reset
speed 50 iter/frame
iterations
0
UCB1 predictability
—
Exp3 predictability
—
RM predictability
—
Press start and watch the three average-strategy charts and the predictability readouts; the speed slider sets iterations per frame. All three algorithms play against themselves in Rock-Paper-Scissors (a symmetric zero-sum game). The lines plot each action's share of all moves so far: the time-average strategy, the quantity the Exp3 and regret-matching convergence guarantees are about. In this symmetric game all three averages settle near the dashed 33.3% Nash line, UCB1's exactly, because its deterministic selection locks into a fixed three-move cycle (the plain-RPS case of Shafiei et al.'s analysis). The strip under each chart shows the last 60 moves, and the predictability stat is the fraction of the last 300 moves that a simple predictor (guess the move that most often followed the previous one) gets right. UCB1 sits at 100%: its cycle can be countered every round. Exp3 and regret matching sample from a mixture, so no opponent can call every move, but their current mixtures keep swinging between near-pure strategies in self-play; the strip shows long runs of one move, and the predictor often scores well above the 33% of truly random play. Only the average converges, the distinction §11 spells out.
Check Your Understanding Q1 of 1

A Smitsimax agent using UCB1 plays Rock-Paper-Scissors for 100,000 rounds against an adversary that knows the agent's selection policy. What happens?
The agent converges to uniform 1/3 and the adversary can't exploit it.
The adversary exploits the agent's deterministic selection to win more than 50% of rounds.
UCB1 adapts and eventually matches the adversary's win rate.
0/1 correct
