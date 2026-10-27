# The task ended, or you stopped watching

Training code cuts episodes short all the time. If a game agent hasn't reached the goal after 200 moves, the loop resets it to collect fresh data. That cutoff is a decision about data collection; the game itself would have gone on. If the code treats it the same as reaching the goal, the target for the last move drops the future term, as if the world had ended there.[3]

So there are two different ways an episode can stop. Termination means the task itself is over: the agent reached the goal, or lost. No more reward can come, so the target drops the future term. Truncation means you stopped watching: a time limit ran out, or a batch filled up. The task would have gone on, so the target keeps γ times the next state's value.

Worked example

The last move earned a reward of 1, the next state's estimated value is 5, and γ = 0.9.

If the task terminated: target = 1.

If a time limit truncated it: target = 1 + 0.9 × 5 = 5.5.

Treat the truncation as a termination and you train that move's value toward 1 instead of 5.5.

One formula covers both cases, with a flag that's 1 only when the task truly ended:
target = reward + γ × (1 − terminated) × next_value

In words: multiply the future term by zero only when the task really ended. A time limit or a batch boundary leaves terminated at 0, so the future term stays.

Gymnasium, a widely used Python interface for RL environments, returns separate terminated and truncated flags for this reason.[4] On truncation, compute the future term from the last observation of the interrupted episode. Vector environments add a trap. In Gymnasium's same-step autoreset mode, the observation returned alongside the truncation already belongs to the next episode, and the real final observation is stored separately. Using the reset observation in the target mixes two episodes together. Check which autoreset mode your environments use, and fetch the real final observation before updating.[5]
Keep or remove the future term
Observed reward
1
Estimated next value
5
Discount γ
0.9
Reset
True terminal target 1.000; external truncation target 5.500 = reward 1 + γ 0.9 × next value 5. Truncation uses the final observation, before reset.

Drag the observed reward, the next-state value estimate and γ. The left bar is the target if the task truly ended: the reward alone. The right bar is the target if a time limit cut the episode short: the reward plus γ times the next value. At the defaults that's 1 versus 5.5. When the deadline is part of the task itself, the state needs to include the time left, as the next paragraph explains.

A deadline that's part of the task is a different case. If the job is "reach the exit within 50 moves", the time left changes which move is best, so it belongs in the state, and running out of time is a real termination. Pardo et al. separate the two cases, which is why "always keep the future term on a time limit" is too broad a rule.[3]
Check Your Understanding Q1 of 2

A task that never ends is paused only because the training batch is full. What should the target do with the future term?
Drop it, because every batch has an end.
Compute it from the reset episode's first observation.
Keep it, using the interrupted episode's final observation.
0/2 correct
