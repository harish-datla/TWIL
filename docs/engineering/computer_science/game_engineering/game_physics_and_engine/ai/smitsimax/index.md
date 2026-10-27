# SMITSIMAX

SMITSIMAX adapts Monte Carlo tree search to simultaneous-move games. Each player maintains an independent search tree, chooses without seeing the other players’ moves, and evaluates the combined simulation separately.

Time: ~50 min  
Level: Intermediate  
Prerequisites: Trees, basic probability, and familiarity with minimax or MCTS.  
Phase: 11 · AI

## What you’ll be able to do

- Write the standard MCTS loop and explain its simultaneous-move failure mode.
- Implement one tree per player with UCB1 selection and joint simulation.
- Tune exploration, search depth, and random-selection thresholds for a turn budget.
- Recognize when non-equilibrium strategies require regret matching instead.
- Correct playout-depth score bias and identify games SMITSIMAX cannot handle.

## Sections

- [Why Simultaneous Moves](01-why-simultaneous-moves.md)
- [A Short History](02-a-short-history.md)
- [Standard MCTS](03-standard-mcts.md)
- [The Simultaneous-Move Problem](04-the-simultaneous-move-problem.md)
- [Decoupled Trees](05-decoupled-trees.md)
- [UCB1 and Selection](06-ucb1-and-selection.md)
- [The Full Algorithm](07-the-full-algorithm.md)
- [Information Sets](08-information-sets.md)
- [Rock-Paper-Scissors Playground](09-rock-paper-scissors-playground.md)
- [Convergence and Nash](10-convergence-and-nash.md)
- [Regret Matching](11-regret-matching.md)
- [Tuning](12-tuning.md)
- [Variable-Depth Trees](13-variable-depth-trees.md)
- [Case Studies](14-case-studies.md)
- [When It Breaks](15-when-it-breaks.md)
- [Pitfalls](16-pitfalls.md)
- [What’s Next](17-whats-next.md)
