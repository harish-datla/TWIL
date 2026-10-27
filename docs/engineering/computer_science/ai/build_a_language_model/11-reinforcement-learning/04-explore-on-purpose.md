# Explore on purpose

An agent that always takes the move it currently rates best can get stuck. Suppose its first lucky trip to the goal went the long way round. That route now has the highest Q values, so the agent repeats it, and it never finds the shortcut because it never tries the moves it rates lower. To learn, it has to spend some moves trying things.

The simplest fix is an ε-greedy policy: with probability ε (epsilon), pick one of the four moves uniformly at random, the current best one included; otherwise take the move with the highest Q value.[1]

Worked example

ε = 0.2, and the Q values for up, right, down and left are 0, 1, 0 and 0, so right is best.

Right: 0.8 from the greedy branch, plus 0.2 / 4 = 0.05 from the random branch, = 0.85.

Each other move: 0.05. Check: 0.85 + 3 × 0.05 = 1.

In general, with four moves and one best move, the best gets 1 − ε + ε/4 and each other move gets ε/4. When several moves tie for best, this widget splits the greedy share, 1 − ε, equally among them. Code that breaks ties another way, say by always picking the first, behaves differently, even though the max in the Q-learning target is the same.
Work out the ε-greedy probabilities
Exploration ε
0.2
Right-action value
1
Reset
Q = [0, 1, 0, 0]. ε=0.2; action probabilities [0.050, 0.850, 0.050, 0.050]; sum 1.000000. Ties split exploitation uniformly.

Drag the right-action value to set how the move right compares with the other three, which stay at 0, and drag ε to set how often the agent explores. Above zero, right is the single best move; at exactly zero all four tie; below zero the other three tie for best. The right panel shows each move's exact probability, random branch included, and they always add up to one. At the defaults (ε = 0.2, right valued at 1) right gets 0.85 and each other move 0.05.

Raising ε gives the low-rated moves more chances. When all four tie, ε changes nothing: every move gets 0.25 either way. Keep ε in mind when you measure an agent, too. Its score while exploring with ε = 0.3 answers a different question from its score when it always takes its best move. With every result, record which policy was evaluated, the random seeds, the episode limit and the reward definition. One successful training episode tells you little about how reliable the agent is.

Random moves are a weak explorer when the reward needs a long, specific sequence of actions, because the chance of stumbling on that sequence falls quickly as it gets longer. How well any exploration works also depends on how the states and actions are represented. For rewards that are rare and far away, treat ε-greedy as the baseline a better method has to beat.
