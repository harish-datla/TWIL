# The full algorithm

Your bot gets a fixed budget each turn, say 50 ms. The loop below runs iterations until the budget is spent; each one clones the current game state, walks every agent's tree one level per simulated turn, plays the combined moves, scores the result and updates each tree with its own agent's score.

Putting it together. The pseudocode below is for k agents in a simultaneous-move game. Each agent has its own tree rooted at a node whose children correspond to that agent's legal moves. The game engine advances the state given a joint action (one move per agent).
Pseudocode

function SmitsimaxSearch(initialState, timeBudget):
    trees = []
    for each agent i in 0..k-1:
        trees[i] = new TreeNode(root=true)

    lowestScore  = [+INF for each agent]
    highestScore = [-INF for each agent]

    while time < timeBudget:
        state = initialState.clone()
        currentNodes = [trees[i] for each agent i]
        for each agent i:
            trees[i].visits += 1   // the root is visited every iteration

        // ── Selection + Expansion + Simulation (to max depth) ──
        for depth in 0..maxDepth-1:
            jointAction = []
            for each agent i:
                node = currentNodes[i]
                if node.visits == 1:     // first arrival: create the children
                    node.expand(state.legalMoves(i))
                // random while node.visits < 10, else UCB1 (see below)
                child = selectChild(node, C, lowestScore[i], highestScore[i])
                child.visits += 1
                currentNodes[i] = child
                jointAction[i] = child.move

            state.advance(jointAction)
            if state.isTerminal():
                break

        // ── Scoring ──
        scores = state.evaluate()  // one score per agent

        // ── Update normalization bounds ──
        for each agent i:
            lowestScore[i]  = min(lowestScore[i],  scores[i])
            highestScore[i] = max(highestScore[i], scores[i])

        // ── Backpropagation ──
        for each agent i:
            node = currentNodes[i]
            while node != null:
                node.totalScore += scores[i]
                node = node.parent

    // ── Choose best move ──
    for each agent i:
        bestMove[i] = trees[i].mostVisitedChild().move
    return bestMove

What's intentionally missing

This pseudocode omits several production concerns: handling variable-length games (some branches terminate early, some agents eliminated before others; covered in §13), tree reuse between turns, parallelization (virtual loss, root parallelism), and domain-specific rollout policies. A real Smitsimax bot also needs fast state cloning and incremental legal-move generation.
Why the original has no separate expansion phase

Classical MCTS tutorials teach a four-phase loop: select down to the first unexplored node, expand it, roll out randomly from there, backpropagate. MSmits's original Smitsimax skips the rollout: every iteration descends all the trees to maxDepth, expanding as it goes.

The decoupled structure shapes that choice. The game state at each depth is determined by the joint action of all agents, so the simulation needs a move from every agent at every depth until maxDepth (or a terminal state). In the original, every move comes from a tree. When an iteration reaches a node for the first time (visits == 1), the node creates its children and the random-selection phase (the parentVisits < 10 guard in selectChild) picks one, so a new branch grows all the way down in a single iteration. Levels where the trees are still thin act like a random rollout that also records statistics; as MSmits puts it, there is no random rollout apart from the first few visits to a node[9].

A conventional rollout also works: after a tree expands one new node, the remaining depth can be filled with random moves for every agent. RoboStac suggested that change, and MSmits reports that it performed a little better in his Coders Strike Back bot[9]; §12 covers the trade-off. The pseudocode above follows the original: the depth loop always runs to maxDepth, and expansion is folded into selection (line if node.visits == 1: node.expand(...)). An iteration ends early only when the game state itself is terminal.
The selection function
Pseudocode

function selectChild(node, C, sMin, sMax):
    parentVisits = node.visits
    if parentVisits < 10:
        return randomChild(node)

    bestScore = -INF
    bestChild = null
    scale = sMax - sMin
    if scale <= 0: scale = 1  // no spread observed yet: avoid dividing by zero

    for each child in node.children:
        if child.visits == 0:
            return child       // try every child once before comparing averages
        exploitation = (child.totalScore / child.visits) / scale
        exploration  = C * sqrt(ln(parentVisits) / child.visits)
        ucb = exploitation + exploration

        if ucb > bestScore:
            bestScore = ucb
            bestChild = child

    return bestChild

Check Your Understanding Q1 of 1

In the Smitsimax algorithm, what happens during backpropagation?
The result propagates up a single shared tree that contains both players' moves.
Each player's tree receives that player's score from the same simulation, propagated independently up the path that tree selected.
Only the winning player's tree is updated; the losing player's tree stays unchanged.
0/1 correct
