# Plan when you know the rules

Sometimes you know the rules completely: for any state and action you can look up the next state and the reward. A board game has fixed rules, and a simulator can tell you what any move does. Then the agent doesn't need to try anything. It can work out how good every state is before taking a single step. That's planning.

Planning rests on a one-step lookahead. A state is worth the best you can do with one move: that move's reward, plus the discounted value of the square it lands on. Applying that rule to one state is a Bellman backup. You don't know the values to start with, so set them all to zero and apply the backup to every square, again and again. Each pass carries news of the goal one more step outward.[1]

Worked example

γ = 0.9 and every value starts at 0.

Sweep 1: square 11 can step down into the goal for +1, and the episode ends there, so there's no future term: V(11) = 1. Square 10 can't reach the goal in one move. Its best option is −0.04 + 0.9 × 0 = −0.04.

Sweep 2: from square 10, moving right lands on square 11, which is now worth 1. V(10) = −0.04 + 0.9 × 1 = 0.86.

Written as a formula, the backup computes a state's new value from the current values of the squares next to it. It's how a planner turns known rules into values.
V_next(s) = max_a [reward(s,a) + γ × V(next(s,a))]

In words: try each of the four moves. Score each one as its reward plus γ times the current value of the square it lands on. The state's new value is the best of those scores. A move into the goal has no future term, and the goal's own value stays at zero. (With known rules in a world with random outcomes, each move's score is an average over the squares it might land on.)

Running the backup over every state, sweep after sweep, is value iteration. The widget's version is synchronous: each sweep reads only the previous sweep's values and writes a fresh array. Updating values in place as you go is a valid variant too, but its in-between numbers differ. A sweep isn't a move. Nobody walks the grid during planning, so don't read the sweep count or the changing arrows as a path the agent takes.[1]
Back up reward through the grid
Discount γ
0.9
Step
Play
Reset
Synchronous sweep 0; discount 0.9; max Bellman residual 1.000000; V(start)=0.0000. Reward +1 entering goal, −0.04 otherwise; goal value zero.

Press Step to run one sweep over every square, or Play to keep sweeping. The left grid shows each square's value; the right grid shows the best move according to those values, with ties going to the first of up, right, down, left. For the first five sweeps the start square's value is slightly negative, because news of the goal hasn't reached it yet. At γ = 0.9 the residual in the readout, the largest change the next sweep would make, shrinks by a factor of 0.9 each sweep and then drops to zero at sweep 6, when the start square settles at 0.4267.

Why does this settle at all? With γ below 1, each sweep shrinks the biggest gap between the current values and the true ones by at least a factor of γ; the residual in the widget shrinks the same way. On any world with a finite number of states, repeated sweeps converge to one set of values.[1] That guarantee covers this exact update with a known model. It isn't a guarantee for a neural network trained on stored experience.
The precise statement

The widget reports the residual instead of promising that some number of sweeps is always enough; a bigger or more tangled world needs more. Planning is also only as good as its model. A perfect solution to the wrong rules still picks the wrong move.
