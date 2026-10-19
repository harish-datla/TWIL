# Learn from answers a program can check

For some tasks a program can check the answer: compare a math result with the known solution, or run the code's tests. Then the reward model goes away. The loop becomes: sample G completions for each prompt from the current policy, score each one, and push up the ones that scored above the rest of their group. GRPO (Group Relative Policy Optimization) scores each completion against its own group: subtract the group's mean reward and divide by the group's standard deviation. That normalized score, the advantage, says how much better than its siblings a completion did. It plays the role of the return minus a baseline on the reinforcement learning page.

Worked example

G = 8 and the reward is 1 for a correct answer, 0 for a wrong one. Two of the eight are correct.

Mean = 2/8 = 0.25. Standard deviation = √(0.25 × 0.75) = 0.433.

Correct completions: (1 − 0.25) / 0.433 = +1.73. Wrong ones: (0 − 0.25) / 0.433 = −0.58.

With one correct answer out of eight the mean is 0.125, the standard deviation 0.331, and the advantages +2.65 and −0.38. The rarer outcome gets the bigger push.

GRPO plugs the advantage into a PPO-style clipped objective, with an optional KL term estimated per token as ratio − log ratio − 1. TRL sets that term's coefficient to zero by default and notes the recent runs that dropped it.[15] RLOO uses the mean of the other G − 1 rewards as the baseline instead, and folds the KL penalty into the reward.[14]
Âi = (ri − mean(r)) / std(r) LGRPO = −(1/Σ|oi|) Σi Σt [ ρi,t·Âi − β·DKL ]

In words: a completion's advantage is its reward minus the group average, divided by the group's spread. The loss weights each token's probability ratio, current policy over the policy that sampled it, by its completion's advantage, subtracts β times a per-token KL estimate, and averages over every token in the group.

Dividing by the spread makes the signal independent of the reward's scale. It also breaks when every completion in a group gets the same reward, because the spread is zero. Implementations set the advantage to zero then, so the group teaches nothing, and TRL logs the fraction of such groups. Dividing by the spread also changes how much hard and easy prompts count, which TRL's documentation flags, with a reference and a switch to turn the division off.[15]
Group-relative advantages and the groups that carry no signal
Group size G
Success probability p
0.8
Seed
1
Reset
Group of 8 completions, each correct with probability 0.8: rewards 1 0 1 1 1 1 1 1, mean 0.875, std 0.331, advantages 0.38 -2.65 0.38 0.38 0.38 0.38 0.38 0.38. Probability that all 8 agree, p^G + (1−p)^G = 0.168.

Pick the group size G and the success probability p; the seed slider draws a different group. Left: one sampled group's rewards, 1 or 0 as a correctness checker gives, and their advantages. At the defaults (G = 8, p = 0.8, seed 1) seven of eight succeed, so each success gets +0.38 and the one failure −2.65. Right: how often a whole group agrees, pG + (1 − p)G, for G = 4, 8 and 16. A group that agrees has zero spread and contributes nothing. At p = 0.9 that's 43% of groups for G = 8 and 66% for G = 4. Prompts the model nearly always solves and prompts it nearly always fails both drop out of training until G is raised or the prompts are filtered.
Check Your Understanding Q1 of 2

RLHF with β = 0.5 and DPO with β = 0.5 start from the same reference model and use the same plentiful, noise-free comparisons. How do the resulting policies relate?
They converge to the same policy: the KL-regularized optimum for the Bradley–Terry reward.
DPO's policy is sharper because it has no KL penalty.
RLHF's policy is better because it uses a reward model.
0/2 correct
