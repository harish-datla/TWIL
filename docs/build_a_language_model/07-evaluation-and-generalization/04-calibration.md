# Calibration

A medical model says a scan has a 90% chance of showing a tumour, and a doctor decides what to do based on that number. For the decision to be sound, the model has to be right about nine times in ten when it says 0.9. That property is calibration, and it's separate from accuracy: a model can rank its cases perfectly while every probability it attaches is too confident.[7]

To check it, group predictions by the probability the model gave, and compare each group's average prediction with how often the event actually happened. Plotted, that's a reliability diagram. The expected calibration error (ECE) squeezes it into one number: the average gap between the two, weighted by how many predictions fall in each group.[7][8]

Worked example

100 predictions in two groups. 40 have an average confidence of 0.9 but only 75% are right: a gap of 0.15. The other 60 average 0.6 and 55% are right: a gap of 0.05.

ECE = 0.40 × 0.15 + 0.60 × 0.05 = 0.09. Both groups claim more than they deliver, so the model is overconfident.

Guo et al. found that modern neural networks trained with cross-entropy tend to come out overconfident.[8] Temperature scaling is a simple repair after training. A classifier turns raw scores called logits into probabilities; temperature scaling divides every logit by one number T, chosen on held-out data. T above 1 pulls the probabilities toward 0.5, or toward equal shares when there are more than two classes. Dividing by a positive number never changes which logit is largest, so every prediction, and the accuracy, stays the same.[8]

Worked example

A binary classifier gives logits 3 and −1. Through the sigmoid, those become probabilities 0.95 and 0.27.

With T = 2 the logits become 1.5 and −0.5, and the probabilities 0.82 and 0.38. Both are less extreme, and both stay on the same side of 0.5, so both predictions are unchanged.

To compare models on probabilities, use a score that rewards being right and being honest about confidence at once. Log loss (the average of −log of the probability given to what actually happened) and the Brier score (the average squared gap between the probability and the 0-or-1 outcome) are proper scoring rules: a model gets its best expected score only by reporting its true beliefs. Accuracy and ECE each measure only one side.[6]
Over-sharpen a calibrated model, then fix it with one temperature
Logit sharpening
2.5
Temperature T
1
Reset
2000 examples whose true probability is sigmoid(z); the model reports sigmoid(2.5·z / T) with T = 1. Accuracy 70.9% (unchanged by T). ECE 0.1286, log loss 0.6641, Brier 0.2054. Temperature minimizing log loss on this set: T = 2.34 (log loss 0.5575).

The bars show, for each probability bin from 0 to 1, the observed fraction of positives (purple) next to the mean predicted probability (green); a calibrated model has matching bars. Drag the temperature T up to the fitted value shown in the readout, then try other sharpening factors. There are 2,000 seeded examples whose labels are drawn from a known probability, so a model that reports that probability is calibrated by construction. Sharpening its logits by 2.5 leaves accuracy at 70.9% but makes its probabilities too extreme in both directions: the purple bars sit above the green ones in the bins below 0.5 and below them in the bins above 0.5. ECE is 0.13 and log loss 0.66. Set T to the fitted value near 2.3 and the bars meet: ECE drops to about 0.02 and log loss to 0.56, with accuracy unchanged. Temperature can't fix a model whose errors aren't a uniform over- or under-confidence.
