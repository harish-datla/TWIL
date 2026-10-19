# Cross-entropy and KL divergence

A language model is trained by showing it real text and, at every position, charging it the surprise of the token that actually came next: minus the log of the probability it gave that token. Averaged over the data, that charge is the cross-entropy, the average surprise of a model q on outcomes drawn from the real distribution p.[2]

Worked example

The truth p is a fair coin, (0.5, 0.5). A model q believes (0.8, 0.2).

Heads costs the model −log₂ 0.8 ≈ 0.32 bits and tails costs −log₂ 0.2 ≈ 2.32 bits. Weighted by how often each really happens: 0.5 × 0.32 + 0.5 × 2.32 ≈ 1.32 bits.

The coin's own entropy is 1 bit, so 0.32 bits of that loss are the model's fault.

That split works for any p and q. Cross-entropy is the entropy of the truth, which no model can reduce, plus the KL divergence from p to q: the extra surprise caused by q differing from p.[2] The formula writes both parts.
CE(p, q) = −Σx p(x) log q(x) = H(p) + KL(p ‖ q)

In words: the model's average surprise on real data equals the data's own unpredictability plus a penalty for the ways the model's probabilities differ from the truth.

KL is never negative, and it's zero only when q matches p.[1] So the lowest possible loss is the entropy of the data, which is above zero whenever the data has any randomness in it. The entropy term doesn't depend on the model, so minimizing cross-entropy on a fixed dataset is the same as minimizing the KL divergence from the data to the model.
Why KL can't go negative

The divergence isn't symmetric. KL(p‖q) and KL(q‖p) usually give different numbers, and they punish different mistakes.

Worked example

p = (0.5, 0.5) and q = (0.9, 0.1).

KL(p‖q) = 0.5 log₂(0.5/0.9) + 0.5 log₂(0.5/0.1) ≈ 0.74 bits.

KL(q‖p) = 0.9 log₂(0.9/0.5) + 0.1 log₂(0.1/0.5) ≈ 0.53 bits.

KL(p‖q) averages over what really happens. If q gives zero probability to something p can produce, the model is infinitely surprised when it occurs, and the divergence is infinite. KL(q‖p) averages over what the model produces, so it doesn't penalize parts of p that q never covers. Maximum likelihood training minimizes KL(p‖q), the direction that punishes a model for ruling out anything the data does.[1][7] The fine-tuning page uses a KL term in a different role, as a leash that keeps a tuned model close to the model it started from.
Fit a model q to a true distribution p
True p
q₁, %
20
q₂, %
20
q₃, %
20
q₄, %
20
Reset
H(p) = 1.3551 bits. Cross-entropy CE(p, q) = 2.3219 bits. KL(p‖q) = CE − H = 0.9668 bits; KL(q‖p) = 1.2503 bits. Both divergences are finite and non-negative.

Purple bars are the true distribution p and green bars are the model q, over five outcomes. Pick a p, then drag q₁ to q₄; q₅ takes the remainder, rescaled if the four pass 100%. At the defaults q is uniform, so CE(p, q) = log₂ 5 ≈ 2.32 bits whatever p is, and the gap above H(p) is KL(p‖q): 0.97 bits for the peaked p. Move the sliders toward p and both divergences fall to zero together. Set a slider to 0 where p is positive and KL(p‖q) becomes infinite while KL(q‖p) stays finite.
