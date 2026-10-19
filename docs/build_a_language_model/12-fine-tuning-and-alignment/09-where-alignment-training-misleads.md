# Where alignment training misleads
A rising reward doesn't mean a better model

In the KL widget, the optimized reward climbs smoothly while the true reward falls below the reference's. A reward model is fit to comparisons of one particular set of outputs, and RL pushes the policy away from that set. Monitor the KL, cap it, and re-evaluate on the policy's own outputs with fresh human or checker judgments. Don't use the reward model that trained the policy as the judge: its mistakes are what the policy may have learned to exploit.

Full fine-tuning on a narrow dataset forgets. In the LoRA widget, the full run ends with a farm loss worse than a uniform guess after 300 iterations on space text. Mixing in pretraining data, lowering the learning rate, or limiting the update's rank all give up some new-domain fit to keep more of the old. Which trade is right depends on whether you still need the old behavior.

Preference labels are noisy and limited in number, and the fit degrades gracefully only up to a point. In the reward widget, 20% flipped labels cost nothing at 400 comparisons and wreck the fit at 20. Compare a reward model's held-out agreement with the ceiling the labels' own noise allows, which the widget puts at 77%, rather than with 100%.

Group-relative methods quietly waste compute on prompts that are too easy or too hard for the current policy. Track the fraction of zero-spread groups, as TRL logs it, and adjust the prompt mix or G when it climbs.[15]
