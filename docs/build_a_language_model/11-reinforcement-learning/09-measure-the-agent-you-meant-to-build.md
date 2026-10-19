# Measure the agent you meant to build

A reward curve mixes together the learning algorithm, the exploration policy, the environment and the way you evaluated. Record each one separately. Compare against a simple fixed policy, run several random seeds, and watch what the agent actually does: a high return can come from exploiting a flaw in the reward instead of doing the task.

Count environment steps and gradient updates separately. Replay can train on the same experience many times, so "steps" on its own is ambiguous. The widgets here count sweeps or sampled moves, and neither one is elapsed game time.
Success on a toy grid doesn't make an agent safe to deploy

The grid shows the agent everything, and nothing in it is random. A real controller can face delayed actions, partial observations and states it never saw in training. Test those conditions directly before relying on a learned policy.
