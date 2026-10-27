# Variable-depth trees

The pseudocode in §7 loops from depth 0 to maxDepth and breaks on terminal states. In real games, branches end at different depths: a snake crashes on turn 3, a unit dies on turn 5, while other branches run to the full depth limit, and some searches (like the fifth-place SnakeByte bot) let the depth grow as the search runs. Two biases follow that the basic algorithm does not handle.
The biases

Score-scale mismatch. Games that accumulate score over time produce larger raw numbers at deeper depths. A branch terminating at depth 3 might score 15; one reaching depth 20 might score 180. Backpropagating raw scores to the same parent averages numbers from incompatible scales. The deep branch dominates the mean even when both strategies perform equally well per turn.

Uneven uncertainty. UCB1 tracks one kind of uncertainty: how many times a node has been visited. It ignores another: how much unexplored tree remains below each child. A branch that always ends the game at depth 3 returns nearly the same result every visit and has little left to learn; a branch above a deep, barely touched subtree has far more, even at identical visit counts. Moerland et al. (2018) formalize this as tree-structure uncertainty and propose a modified UCB formula that shrinks the exploration bonus for fully explored subtrees[19].
Live · Depth-bias correction
run
reset
Siege avg depth 15
iterations
0
raw best
—
corrected best
—
Press run to sample rollouts for all three actions, then drag the slider and run again. Three actions whose rollouts are scored at different depths: Sprint's after 2–5 turns, March's after 6–10, Siege's after 13–18, with no action ending the game early. Sprint earns the most points per turn but the fewest total points. Raw averages (left) make Siege look about 3× better than Sprint only because its rollouts run longer. Per-turn normalization (right) removes the depth scaling and ranks Sprint first. Drag the slider to lengthen Siege's rollouts: the raw bias grows while the corrected ranking stays put. If a branch is short because the agent died, the short length is itself the outcome, and dividing by depth would hide it; see the note below.

Worked example

In the widget's default setup, Sprint rollouts last 2 to 5 turns (3.5 on average) and score 4.0 to 5.5 points per turn (4.75 on average), so a Sprint rollout averages about 3.5 × 4.75 ≈ 16.6 points. Siege rollouts last 13 to 18 turns (15.5 on average) at 2.8 to 3.6 points per turn (3.2 on average), about 49.6 points. Raw averages rank Siege about 3× ahead. Per turn, Sprint earns 4.75 and Siege 3.2, so Sprint is the better action.
Depth-discounted backpropagation

A common fix, borrowed from reinforcement learning: apply a discount factor γ (0 < γ < 1) during backpropagation. Each step from leaf toward root multiplies the score by γ.
Pseudocode · Discounted backpropagation

function backpropagate(currentNodes, scores, gamma):
    for each agent i:
        discountedScore = scores[i]
        node = currentNodes[i]
        while node != null:
            node.totalScore += discountedScore
            discountedScore *= gamma  // each step toward root attenuates the signal
            node = node.parent

With γ = 0.97, a score evaluated at depth 10 reaches the root multiplied by 0.9710 ≈ 0.74. A score at depth 3 is multiplied by 0.973 ≈ 0.91. Near-term outcomes propagate more strength to the root than distant ones, much as the discount factor in reinforcement learning makes later rewards count less.

Tradeoff: γ < 1 biases toward short-sighted play. If the optimal strategy requires sacrificing short-term score for a large payoff at depth 15, discounting undervalues that line. Values just below 1 (0.95 to 0.99) damp the depth effect without discarding the long-range signal; tune γ like C, per game.
Depth-normalized evaluation

Instead of (or alongside) discounting, normalize the evaluation output so that scores at different depths are comparable before backpropagation.
Pseudocode · Depth-normalized scoring

function evaluateNormalized(state, currentDepth):
    rawScores = state.evaluate()
    for each agent i:
        // Convert to per-turn value: removes depth-proportional scaling
        rawScores[i] = rawScores[i] / max(currentDepth, 1)
    return rawScores

For games with cumulative scoring, dividing by depth converts raw score to "score per turn," which no longer grows with depth when points accrue at a steady rate. Use it where the depth differences come from the search (a varying or growing depth limit), not from the game: if a branch is short because the agent was eliminated, per-turn scoring can make an early death look as good as survival. Score that outcome directly in the terminal evaluation instead (placement, survival time, a large penalty for dying). For games with a known maximum achievable score at each depth, normalizing by that maximum maps everything to [0, 1]. This interacts with the score normalization from §12: compute the Smin/Smax bounds over normalized scores, not raw scores, otherwise the running bounds will track the depth-scale difference rather than the strategic difference.
Agent elimination

In multi-agent games, agents get eliminated at different points. A dead agent can't act, but the game continues for surviving agents. Two approaches for the dead agent's tree:

Pass-through nodes. When an eliminated agent's tree is descended during an iteration, select a special "pass" child at each depth beyond the elimination point. The pass child carries no action; the game state advances using only the surviving agents' moves. Statistics still accumulate on the pass child, because the dead agent's tree needs to learn whether dying at that point led to a good or bad final outcome (e.g., 1st place vs. 4th).
Pseudocode · Selection with agent elimination

// Inside the depth loop from §7:
for each agent i:
    node = currentNodes[i]
    if not state.isAlive(i):
        // Dead agent: descend through a pass node, contribute no action
        child = node.getOrCreatePassChild()
        jointAction[i] = NO_ACTION
    else:
        if node.visits == 1:
            node.expand(state.legalMoves(i))
        child = selectChild(node, C, lowestScore[i], highestScore[i])
        jointAction[i] = child.move
    child.visits += 1
    currentNodes[i] = child

Freeze the tree. Stop selecting and expanding the dead agent's tree entirely after elimination. Simpler to implement, but the agent's tree cannot learn from post-death outcomes. If scoring depends on final placement or team result, pass-through nodes are more informative.
Proven terminals

When a simulation reaches a terminal state (game over), that result is exact, not a statistical estimate. Winands, Björnsson, and Saito (2008) introduced MCTS-Solver[18], which propagates proven game outcomes up the tree as hard values rather than folding them into the running average. A node is proven a win for the player to move if any child is a proven win, and a proven loss if every child is; proven nodes stop being sampled and report their exact value.

In Smitsimax, proven results are harder to establish than in sequential games. A node in one player's tree maps to an information set, not a single game state (§8). A terminal outcome in one iteration may not recur in the next because the opponent's concurrent selections lead to a different game state at the same tree node. A node can only be marked proven if the outcome holds across all possible opponent paths. The practical application is narrower: if an agent's tree consistently reaches terminal death at a given depth regardless of opponent play, penalize that subtree during selection to avoid wasting iterations on it.
Check Your Understanding Q1 of 1

Two rollouts through the same Smitsimax parent node are cut off by the search at different depths: rollout 1 stops at depth 3 with score 12, rollout 2 at depth 15 with score 60. Neither line ended the game, and both scored 4 points per turn. The raw average is 36. What is the problem?
Nothing. The raw average correctly reflects that rollout 2 found a more valuable line.
The two scores are on different scales. The raw average over-weights the deeper rollout because it accumulated more points over more turns, not because the strategy was better.
The parent should discard rollout 1 because it did not search deep enough to be informative.
0/1 correct

For a broader treatment of backup operators, Dam et al. (2020) show that replacing the arithmetic mean with a power mean (exponent p > 1) during backpropagation reduces the negative bias inherent in average-based backup[20]. The paper addresses MCTS backup generally, not variable-depth trees specifically; whether the power mean helps with mixed-depth evaluations is something to measure per game. The power mean is defined for non-negative values, so scores need normalizing first (the [0, 1] scaling from §6 and §12 works), and it adds one tuning parameter.
