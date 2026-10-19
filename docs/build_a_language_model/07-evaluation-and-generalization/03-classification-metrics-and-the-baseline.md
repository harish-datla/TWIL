# Classification metrics and the baseline

A defect detector on a factory line reports 99% accuracy. If 99% of parts are fine, a detector that never flags anything scores the same. Accuracy, the share of all predictions that are right, gets fooled whenever one class dominates. The fix is to count the two kinds of mistake separately and to compare every score with a trivial baseline.

Most classifiers output a score, and the score becomes a yes-or-no decision only once you pick a threshold. For one threshold, the confusion matrix counts four outcomes: true positives (flagged and really positive), false positives (flagged but negative), false negatives (missed positives) and true negatives. Three metrics come from it. Precision is the share of flagged examples that are really positive. Recall is the share of real positives that got flagged. F1 combines the two and is low when either is low.[6]

Worked example

100 emails, 10 of them spam. The filter flags 12, and 8 of those are spam. So TP = 8, FP = 4, FN = 2, TN = 86.

Precision = 8/12 ≈ 0.67. Recall = 8/10 = 0.80. F1 = 2 × 0.67 × 0.80 / (0.67 + 0.80) ≈ 0.73. Accuracy = (8 + 86)/100 = 94%.

A filter that never flags anything gets 90% accuracy, so 94% is only 4 points above doing nothing.

That trivial predictor is the baseline to report next to every accuracy. scikit-learn ships "dummy" classifiers, such as one that always predicts the most frequent class, for exactly this sanity check.[6] The three formulas, which change whenever the threshold moves:
P = TP / (TP + FP),   R = TP / (TP + FN),   F₁ = 2PR / (P + R)

In words: precision asks "when it says yes, how often is it right?", recall asks "of everything it should have found, how much did it find?", and F1 is a single score that stays low unless both are high.

Raising the threshold usually buys precision at the cost of recall, so a single threshold's numbers don't describe the classifier as a whole. Sweep the threshold from high to low, plot the share of positives found (the true-positive rate, which is recall) against the share of negatives wrongly flagged (the false-positive rate), and you get the ROC curve. The area under it, the AUC, has a direct meaning: the probability that a randomly chosen positive gets a higher score than a randomly chosen negative. It measures how well the scores rank examples, with no threshold involved.[6]

Worked example

Two positives score 0.9 and 0.4; two negatives score 0.6 and 0.2. That makes four positive-negative pairs: 0.9 beats 0.6, 0.9 beats 0.2, 0.4 loses to 0.6, 0.4 beats 0.2.

Three of four pairs are ranked correctly, so AUC = 0.75. A classifier that scores at random gets 0.5.
Move the threshold, change every metric but one
Positive fraction
Class separation
2
Threshold
1
Reset
600 examples, 299 positive. Threshold 1: TP 243, FP 47, FN 56, TN 254. Precision 83.8%, recall 81.3%, F1 0.825, accuracy 82.8% against a majority-class baseline of 50.2%. ROC AUC 0.9148 (trapezoid) = 0.9148 (rank statistic).

Left: histograms of the scores for negatives (purple) and positives (green), with the threshold in yellow. Right: the ROC curve, with the current threshold as a yellow dot. Drag the threshold, then change the class balance. The 600 scores are seeded: negatives come from a normal distribution centred on 0, and positives are shifted right by the separation. Moving the threshold trades precision for recall and slides the yellow dot along the ROC curve, but the AUC doesn't move. The readout computes the AUC two ways, as the area under the curve and as the fraction of positive-negative pairs ranked correctly, and they agree. Switch to 10% positives: at threshold 1 the accuracy is 85.0% while the always-negative baseline is 90.7%, so the classifier is below the baseline on accuracy even though its recall is 84% and its precision 37%. Accuracy is the wrong summary for that problem.
Check Your Understanding Q1 of 2

A pipeline standardizes features using the mean and variance of the whole dataset, then splits off a test set. Why is the test score optimistic?
Standardization changes the labels.
The scaler's statistics include the test examples, so information about them reached the fit.
It is not optimistic; scaling is label-free and cannot leak.
0/2 correct
