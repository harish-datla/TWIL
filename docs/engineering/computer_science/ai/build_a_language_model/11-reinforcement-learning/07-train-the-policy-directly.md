# Train the policy directly

Everything so far learned values and picked moves from them. Another family of methods skips the values and adjusts the policy itself. The policy becomes a function with weights that outputs a probability for each action, and training makes the actions that led to high return more likely. The next page uses this kind of method on language models, where the policy is the model and each action is the next token.

After trying an action and seeing its return, nudge the weights in the direction that raises that action's log probability, scaled by the return. That direction is the gradient of log π, where π is the policy's probability for the action. This recipe is a score-function estimator.[6]

It's noisy. If every action earns a positive return, every sampled action gets pushed up, just by different amounts, and the estimate swings a lot from one sample to the next. The standard fix is to subtract a baseline from the return: a reference level, such as the return you'd expect in this state. Actions that beat it get pushed up and actions that fall short get pushed down.

The widget uses the smallest possible case, a two-action bandit: one decision, with no later states. Action 0 always pays 1 and action 1 always pays 4. The policy has a single weight, a logit, and picks action 1 with probability p. The gradient of log probability with respect to that logit is 1 − p for action 1 and −p for action 0.

Worked example

p = 0.35, so the gradients of log probability are 1 − 0.35 = 0.65 for action 1 and −0.35 for action 0.

No baseline: picking action 1 gives 4 × 0.65 = 2.6; picking action 0 gives 1 × (−0.35) = −0.35. Average: 0.35 × 2.6 + 0.65 × (−0.35) = 0.6825.

Baseline 2.95: action 1 gives (4 − 2.95) × 0.65 = 0.6825 and action 0 gives (1 − 2.95) × (−0.35) = 0.6825. The average is unchanged, and both samples are now equal, so the noise is gone.

The general estimator, for one sampled action, is:
estimate = (return − baseline) × ∇ log π(action | state)

In words: take how much the sampled action's return beat the baseline, and multiply by the direction that makes that action more likely. Averaged over many samples, this points in the direction that raises expected return. The baseline mustn't depend on which action was sampled, and it's held constant when you take the gradient.

Why doesn't the baseline change the average? Averaged over actions, its contribution is the baseline times Σ π(a|s) ∇log π(a|s). Since π ∇log π = ∇π, that equals the baseline times ∇ Σ π(a|s). The probabilities always add up to 1, and the gradient of a constant is zero. The variance can still go either way, and a poorly chosen baseline can make it worse. A learned estimate of the state's value is a common baseline, but it isn't guaranteed to give the lowest variance.[6]
Move the baseline: same average, different noise
Probability of action 1
0.35
Baseline
0
Reset
Gradient samples for actions 0 / 1: -0.350000 / 2.600000. Exact expected logit gradient 0.682500; estimator variance 1.979819. For this deterministic two-action bandit, variance-minimizing baseline 2.950; rewards 1 and 4.

Drag the baseline and compare the panels. The left bars are the gradient sample you'd get if action 0 or action 1 were picked; the right bars are their exact average and variance, computed from both outcomes instead of by sampling. At the default p = 0.35 the average stays at 0.6825 for every baseline, while the variance falls from 1.98 at baseline 0 to zero at 2.95. A baseline that removes all the variance exists here only because there are two actions with fixed rewards; in general no baseline does that.

A full multi-step policy gradient also has to decide how discounting enters and which states count in the objective. The one-decision bandit has none of that, so the widget shows the baseline's effect on its own.
