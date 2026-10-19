# Softmax, temperature and log-sum-exp

A classifier's last layer, or a language model's, produces one raw score per class or per vocabulary token. These scores, the logits, can be any real numbers, so they aren't probabilities yet. The softmax turns them into probabilities: raise e to the power of each score, which makes every value positive, then divide by the total so they add up to 1. A bigger logit always gets a bigger probability, so the top class doesn't change.[3]

When a language model generates text, it samples each next token from this distribution, and a temperature T controls how adventurous the sampling is. Every logit is divided by T before the softmax. A temperature below 1 widens the gaps between logits and sharpens the distribution toward the top choice. Above 1 it narrows the gaps and flattens the distribution toward uniform.[3]

Worked example

Logits (2, 1, −1). e² ≈ 7.389, e¹ ≈ 2.718, e⁻¹ ≈ 0.368, total ≈ 10.475.

Probabilities: 7.389 / 10.475 ≈ 0.705, then 0.259 and 0.035.

At T = 0.5 the same logits give (0.879, 0.119, 0.002). At T = 2 they give (0.547, 0.331, 0.122).

Exponentials overflow fast. A double tops out near 1.8 × 10³⁰⁸, so ez is infinite for any z above about 709.8, and a logit of 1000 makes the softmax divide infinity by infinity, which gives NaN. The fix uses the fact that adding the same number to every logit doesn't change the softmax: subtract the largest logit first. Every exponent is then 0 or below, so every exponential is at most 1.

Worked example

Logits (1002, 1001, 999). Subtract 1002 to get (0, −1, −3).

Subtracting 2 from (2, 1, −1) gives the same (0, −1, −3), so the probabilities are the same 0.705, 0.259 and 0.035, and nothing overflowed.

The formula gives the softmax with temperature, then the form frameworks compute in practice: the log of each probability, written with the log-sum-exp (LSE). That's the log of the sum of the exponentials, computed with the largest value subtracted.
pi = exp(zi/T) / Σj exp(zj/T),   log pi = zi/T − LSE(z/T)

In words: each class's probability is its exponentiated, temperature-scaled logit divided by the sum of those over all classes. The log of that probability is just the scaled logit minus one shared number, the log-sum-exp, so the code never forms a tiny probability and then takes its log.

PyTorch's cross-entropy loss takes logits, not probabilities, and is documented as a log-softmax followed by negative log-likelihood.[8]

For a classifier, the truth for one example is a single class: a one-hot vector y with 1 for the correct class and 0 everywhere else. Cross-entropy against a one-hot target keeps only one term, so the loss is −log p of the correct class. Its gradient with respect to the logits is p − y, the predicted probabilities minus the one-hot target.[3]

Worked example

Logits (2, 1, −1) with the first class correct: loss = −ln 0.705 ≈ 0.349 nats.

Gradient p − y = (0.705 − 1, 0.259, 0.035) = (−0.295, 0.259, 0.035).

Gradient descent steps against the gradient, so it raises the correct logit and lowers the other two, each in proportion to its probability.

The loss is never below zero, and it only reaches zero as the correct class's probability approaches 1, which needs infinitely large gaps between logits. So training keeps pushing a confident classifier's logits further apart.[3] A wrong answer given with probability near 1 is expensive: the correct class then has probability near zero, and −log of a number near zero has no upper limit.
Scale three logits by a temperature and read the loss and gradient
z₁
2
z₂
1
z₃
-1
Temperature T
1
True class
Reset
Logits (2, 1, -1) / T=1 → probabilities (0.7054, 0.2595, 0.0351); log-sum-exp = 2.3490. True class 1: loss −log p = 0.3490 nats. Gradient of the loss with respect to the scaled logits z/T, p − y = (-0.2946, 0.2595, 0.0351). Adding 1000 to every logit changes the probabilities by at most 1.0e-15.

The left bars are the softmax of the three logits, with a yellow box on the true class. The right bars are the gradient of the loss for each logit. Drag the temperature: lowering it sharpens the distribution toward the largest logit, and raising it flattens it toward uniform, where the loss approaches log 3 ≈ 1.099 nats whatever the logits. The gradient bars, taken with respect to the scaled logits z/T, sum to zero: the true class is pushed up by 1 − p and every other class down by its own probability. The gradient with respect to the raw logits is the same vector divided by T. The readout also recomputes the probabilities after adding 1000 to every logit; the stable log-sum-exp gives the same answer, where a naive exp would overflow.
Why it's called temperature

Check Your Understanding Q1 of 2

A language model reaches a loss equal to the entropy of its evaluation text. What is left to gain?
Nothing on that text: the cross-entropy cannot go below the entropy of the data.
Half the loss, with a bigger model.
It can reach zero loss by memorizing the text.
0/2 correct
