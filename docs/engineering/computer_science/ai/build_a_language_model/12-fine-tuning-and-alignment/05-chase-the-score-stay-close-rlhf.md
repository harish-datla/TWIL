# Chase the score, stay close: RLHF

With a reward model in hand, you can train the language model to produce replies the reward model scores highly. The model being trained is called the policy, as on the reinforcement learning page, and this stage is RLHF, reinforcement learning from human feedback. Left alone, the policy can find replies the reward model overrates, because the reward model only learned from the kinds of replies in its comparison data. So RLHF charges for drifting.

The objective is the expected reward minus β times the KL divergence from the reference policy, the model you started from. KL divergence measures, in nats, how far one probability distribution has moved from another, and it's zero when they're equal (Probability for ML covers it). Over a finite set of replies, the best policy has a simple form: take the reference's probabilities, multiply each by ereward/β, and rescale so they add up to 1.

Worked example

Two replies, each with reference probability 0.5. Reply 1 has reward 1, reply 2 has reward 0.

β = 0.5: weights 0.5 × e2 = 3.69 and 0.5 × e0 = 0.5, so the best policy gives reply 1 0.88 and reply 2 0.12.

β = 2: weights 0.5 × e0.5 = 0.82 and 0.5, so reply 1 gets only 0.62. A larger β keeps the policy closer to the reference.

The objective and the best policy it leads to are:
maxπ Ey∼π[r(y)] − β·KL(π ‖ πref) ⇒ π*(y) = πref(y)·exp(r(y)/β) / Z

In words: choose the policy that maximizes average reward minus β times its KL divergence from the reference. The best such policy is the reference reweighted by er(y)/β and divided by Z, the sum of those weights, so the probabilities add up to 1. A small β lets high-reward replies take over; a large β keeps the policy near the reference.
Where the closed form comes from

OpenAI's 2019 code for fine-tuning language models from human preferences applies the penalty per token. Each generated token's reward gets −kl_coef·(log π − log π_ref), the reward model's score is added at the final token, and PPO, a policy-gradient method that keeps each update small, maximizes the sum.[12] The coefficient was 0.15 with an adaptive target of 6 nats for stylistic tasks and 0.01 to 0.03 for summarization, with a note in the launch script that 0.01 was too low.[13]
Optimize a policy over six responses against a reward model that is wrong about one
β
0.5
Reward
Step
Play
Reset
β = 0.5, 0 exact gradient steps on E[reward] − β·KL. Policy over A…F: 0.35 0.25 0.15 0.12 0.08 0.05; closed-form optimum 0.00 0.00 0.01 0.02 0.03 0.94 (largest gap 8.9e-1). KL to the reference -0.00 nats. Expected reward under the optimized reward 0.82; under the true reward 0.56 (reference policy: 0.56).

Drag β, choose which reward the policy optimizes, and press Step (ten exact gradient steps, so no sampling noise) or Play. There are six candidate replies to one prompt, and the reference policy gives them probabilities from 0.35 down to 0.05. The true rewards are 0, 0.5, 1, 1.5, 2 and −1, so reply F is bad. The reward model agrees on A to E but scores F at +4, the kind of mistake a reward model can make on a reply it rarely saw in its comparison data; F is the rarest reply under the reference. At β = 0.5, after 100 steps the policy puts 94% of its probability on F: the optimized reward has climbed from 0.82 to 3.83, the true reward has fallen from 0.56 to −0.86, and the KL is 2.6 nats. At β = 2 the policy stays within 0.2 nats of the reference and F gets 21%, but the true reward, 0.49, is still below the reference's 0.56. Switch to the true reward to see what the penalty buys when the reward is right: at β = 0.5 the true reward reaches 1.54 at 0.9 nats. Over six replies the gradient steps land on the closed-form optimum every time, which checks the update; a language model has no such check.

Two more parts of the released implementation matter in practice. Rewards are whitened, scaled to unit variance across the batch, before PPO sees them. And a KL controller adjusts the coefficient after each batch, multiplying it by 1 + clip(KL/target − 1, −0.2, 0.2)·n/horizon, so a run that drifts too fast is charged more.[12] Later trainers moved the penalty around. RLOO folds it into the scalar reward with no gradient through it, and uses the mean reward of the other completions as a baseline: a reference level subtracted from each reward to cut noise, as on the reinforcement learning page. GRPO adds the penalty to the loss as a term the gradient flows through.[14][15]
