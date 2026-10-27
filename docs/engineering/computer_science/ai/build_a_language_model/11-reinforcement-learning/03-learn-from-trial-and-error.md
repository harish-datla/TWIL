# Learn from trial and error

Usually you don't have the rules. A robot can't look up what its wheels will do on a wet floor, and a game-playing agent might see only the screen and never the game's code. What the agent can always do is act and watch. Each move gives it one record: it was in state s, took action a, got reward r, and landed in state s′. Q-learning learns action values from those records alone.[1]

It keeps a table with one number per square and action, Q(s, a): its current guess at the return for taking action a in square s. After each move it forms a better guess from what just happened: the reward it actually got, plus γ times the best Q value at the square it landed on. Then it moves the old guess part of the way toward the better one. The future part of that better guess comes from the table itself, so it's called the bootstrap term: the table uses its own estimates to improve its estimates.

Worked example

The agent is in square 10 and Q(10, right) = 0. It moves right to square 11 and gets −0.04. The best Q value at square 11 so far is 0.5. γ = 0.9 and the learning rate α (alpha) is 0.4.

target = −0.04 + 0.9 × 0.5 = 0.41. error = 0.41 − 0 = 0.41.

new Q(10, right) = 0 + 0.4 × 0.41 = 0.164. No other entry in the table changes.

The whole update is three lines, and it's how the agent turns one experienced move into a better value guess.
target = reward + γ × max_a Q(next_state,a)
error = target − Q(state,action)
Q(state,action) ← Q(state,action) + α × error

In words: the target is the reward you saw plus the discounted best guess for where you landed. The error is how far the old guess was from the target. The learning rate α says what fraction of that gap to close. When the move truly ends the episode, the future term is dropped and the target is just the reward.

That error is the temporal-difference error, or TD error, because it compares guesses made one time step apart. A supervised label stays fixed. This target is built from the table, so it moves as the table learns. Each update changes only the one entry for the state and action just taken.
Learn from one move at a time
Exploration ε
0.25
Discount γ
0.9
Step
Play
Reset
0 sampled transitions; 0 goals reached; current state 0; latest TD error 0.0000. α=0.4, seed=1977. Accumulated undiscounted reward 0.00.

Press Step to take one move and make one update, or Play to keep going. The left grid shows the best Q value in each square, with the agent's square in yellow; the bars on the right are the four Q values where the agent stands now. Values go negative at first, because every move costs −0.04 and the +1 only starts spreading back once the agent stumbles into the goal. Reaching the goal sends the agent back to square 0. Changing a slider restarts the run with the same random seed.

Q-learning is off-policy: the target always uses the best next action (the max), even when the agent will actually explore and take a different one. So the agent can behave one way, with random exploring moves, while learning the values of a different policy that always takes the best move. That doesn't let it skip experience it never had. A state and action the agent never tries never gets an update, so its value stays a guess.[1]

The widget uses a fixed learning rate of 0.4 so each change is big enough to see.
When is Q-learning guaranteed to converge?

Check Your Understanding Q1 of 2

Why can Q-learning update a value without knowing the rules of the world?
A neural network predicts every transition exactly.
It ignores future rewards.
It builds its target from one move it actually experienced.
0/2 correct
