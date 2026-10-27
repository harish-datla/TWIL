# Skip the reward model: DPO

RLHF is two big jobs: train a reward model, then run RL against it, generating fresh samples from the policy as it trains. DPO (Direct Preference Optimization) targets the same optimum with one supervised-style training run on the comparisons. It runs the closed form backwards.

If the best policy is the reference reweighted by ereward/β, then the reward can be read off the policy: r(y) = β·log(π*(y)/πref(y)) + β·log Z. Put that into the Bradley–Terry probability. The β·log Z terms cancel, because both replies in a comparison answer the same prompt and so share the same Z. What's left is a classification loss that mentions only the policy and the reference, with no reward model and no sampling.[16][17]

Worked example

β = 0.5. For one comparison, training has raised the policy's probability of the chosen reply from 0.10 (its reference probability) to 0.20, and lowered the rejected reply's from 0.10 to 0.05.

Log ratios: ln(0.20 / 0.10) = 0.693 and ln(0.05 / 0.10) = −0.693. The margin inside the σ is 0.5 × (0.693 − (−0.693)) = 0.693.

Loss: −ln σ(0.693) = −ln(2/3) = 0.405. Before training both ratios were 1, the margin was 0, and the loss was ln 2 = 0.693. The implicit rewards are now 0.5 × 0.693 = 0.35 for the chosen reply and −0.35 for the rejected one.

The DPO loss for one comparison, and the reward it implicitly learns, are:
LDPO = −log σ( β·( log πθ(y⁺)/πref(y⁺) − log πθ(y⁻)/πref(y⁻) ) ) r̂(y) = β·log(πθ(y)/πref(y))

In words: for each comparison, measure how much more the policy has raised the chosen reply's probability, relative to the reference, than the rejected reply's. Scale that by β and apply the same −log σ loss as the reward model. β·log(π/πref) is the reward DPO has learned without ever fitting one, which is why TRL logs it.
What the DPO gradient does

The reference code adds label smoothing for noisy labels, which mixes in the loss of the flipped pair with weight ε, the assumed fraction of flipped labels. Its README suggests β between 0.1 and 0.5 as a starting range, and running SFT first so the comparison pairs look like the model's own outputs.[16][18]
Run DPO on sampled comparisons and compare it with the RLHF optimum
β
0.5
Comparisons
Labels flipped
Step
Play
Reset
β = 0.5, 1000 comparisons (0% flipped), 0 gradient steps. DPO policy over A…F: 0.35 0.25 0.15 0.12 0.08 0.05; the KL-regularized optimum for the true reward at this β: 0.04 0.08 0.12 0.27 0.49 0.00 (largest gap 0.410). Implicit rewards β·log(π/π_ref), centered: 0.00 0.00 0.00 0.00 0.00 0.00 against true rewards centered -0.67 -0.17 0.33 0.83 1.33 -1.67. Pair accuracy 0.000.

Drag β, pick the number of comparisons and the share of flipped labels, and press Step (25 gradient steps on the pairs) or Play. These are the same six replies and reference as above. The comparisons are labelled from the true rewards through Bradley–Terry, and no reward model is fit. Left: the reference, the DPO policy and the closed-form optimum for the true reward. Right: the implicit rewards β·log(π/πref) against the true rewards, both centered. The policy's logits start at the reference and move only where pairs push them. After 1,000 steps on 1,000 pairs the ordering is right and the largest probability gap to the optimum is 0.04. With 200 pairs, about 13 for each pair of replies, the sampled labels put D above E and the gap is 0.22; with 50 the ordering is wrong in several places. Flipping 20% of labels squeezes the implicit rewards together, as it squeezed the reward model's scores, and the policy stays closer to the reference. β plays the same role as in RLHF: at β = 0.2 the 1,000-pair policy puts 84% on E, the best reply under the true reward, against 45% at β = 0.5.
