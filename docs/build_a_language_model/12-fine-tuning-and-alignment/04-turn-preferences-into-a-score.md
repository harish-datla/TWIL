# Turn preferences into a score

Example answers show the model what to say. Preference data asks people a different question: of two replies, which is better? A preference example is a prompt with a chosen and a rejected completion. That's the format TRL's trainers take, and the format of the HH-RLHF and summarization datasets.[10][4][3]

A reward model turns those comparisons into a score for any single reply. The link TRL's reward trainer uses between scores and choices is the Bradley–Terry model: the bigger the score gap between two replies, the more likely people pick the higher-scoring one. The probability is the sigmoid of the gap, where the sigmoid, 1/(1 + e−z), squashes any number into a probability between 0 and 1. Training adjusts the scores to make the observed choices as likely as possible.[11]

Worked example

Reply A scores 1.0 and reply B scores −0.5, a gap of 1.5.

P(A preferred) = σ(1.5) = 1 / (1 + e−1.5) ≈ 0.818.

If the person picked A, the loss on that comparison is −ln 0.818 ≈ 0.201. If they picked B, it's −ln(1 − 0.818) ≈ 1.701. Add 10 to both scores and the gap, and so every number here, stays the same.

The probability and the loss for one comparison are:
P(y⁺ ≻ y⁻ | x) = σ(rθ(x, y⁺) − rθ(x, y⁻)) LRM = −log σ(rθ(x, y⁺) − rθ(x, y⁻))

In words: the probability that people prefer y⁺ over y⁻ for prompt x is the sigmoid of the reward model's score gap, and the loss is minus the log of that probability. Training pushes the chosen reply's score up and the rejected reply's score down.

Only score differences appear, so adding the same constant to every score leaves the loss unchanged. Nothing in the data pins down the average score, and TRL offers a penalty on the mean score to hold it in place.[11]
Fit Bradley–Terry scores to sampled comparisons
Comparisons
Labels flipped
Centering penalty
Reset
100 comparisons among 8 responses, 0% of labels flipped, scores initialized at 2 with the centering penalty. Fitted scores correlate 0.901 with the true ones; mean fitted score 0.013. Agreement with 4,000 fresh comparisons, none flipped: 0.747, against 0.770 for the true scores themselves (labels are themselves random draws, so no scorer reaches 1). Agreement on the training pairs: 0.770.

Pick the number of comparisons and the share of flipped labels, and toggle the centering penalty. Eight responses have hidden true scores from +1.8 to −1.5. Each comparison picks two at random and labels the winner by sampling from the Bradley–Terry probability of the true scores, so even the true scores agree with only 77% of fresh labels: no scorer can beat the noise in the labels. Left: true scores (purple) and fitted ones (green). Right: the training loss at each step. With 100 clean comparisons the fitted scores correlate 0.90 with the truth and agree with 75% of fresh labels; 400 comparisons reach 0.99 and the 77% ceiling; 20 give 0.51 and 65%. Flipping 20% of labels mostly squeezes the fitted scores together and costs no agreement at 400 comparisons, but ruins the fit at 20. Scores start at 2 to make the free constant visible: the comparison loss never moves the mean, so without the centering penalty the fitted mean stays at exactly 2, and with it (coefficient 0.1 on the squared mean here) the mean is pulled to 0.01 within the 400 steps.
