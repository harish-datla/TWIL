# Measuring wrongness: loss functions

A freshly built network has random weights and makes random predictions. To improve it, training needs one number that says how wrong it is across the training examples, so it can work out which way to nudge each weight to make that number smaller. That number is the loss (also called the cost). Lower is better, and a perfect prediction scores zero.
Mean squared error

The simplest loss, used in Nielsen's book[4] and many introductory examples, is mean squared error (MSE): measure how far each prediction is from its target, square that distance, and average over the examples. Squaring makes every error positive and punishes big misses more than small ones.

Worked example

Target y = (1, 0): the example belongs to class 1 of 2. Prediction ŷ = (0.8, 0.3).

Squared distance: (1 − 0.8)² + (0 − 0.3)² = 0.04 + 0.09 = 0.13. With the ½ factor from the formula below, this example's loss is 0.065.

Written out for n examples, with true output y and prediction ŷ for each:
eq. 6 · MSE C = 12n ∑x ‖y(x) − ŷ(x)‖2

In words: for each training example, take the distance between the prediction and the truth, square it, add those up, and divide by twice the number of examples. The ½ is there for convenience: it cancels when you take the slope, which comes out as (ŷ − y) instead of 2(ŷ − y).

Cross-entropy: the better choice for classification

MSE works, but when the network is choosing between classes (spam or not, cat or dog, which word comes next) the usual loss is cross-entropy. Instead of measuring numeric distance, it asks how much probability the model gave the correct answer, and charges a lot when that probability is small.

That needs probabilities first. The output layer's raw numbers, called logits, can be any size and sign. Softmax makes each one positive by raising e to its power, then divides each by the total so they add up to 1.

Worked example

Logits (2.0, 0.5). Exponentiate: e2.0 ≈ 7.389 and e0.5 ≈ 1.649. Total ≈ 9.038.

Probabilities: 7.389 / 9.038 ≈ 0.818 and 1.649 / 9.038 ≈ 0.182. The bigger logit gets the bigger share, and the shares add to 1.

If class 1 is correct, the cross-entropy loss is −log(0.818) ≈ 0.201.

Softmax computes those probabilities for logit i out of all the logits j:
eq. 7 · softmax softmax(z)i = ezi∑j ezj

In words: raise e to the power of each logit, which makes every one positive, then divide each by the total of all of them. The result is a list of probabilities that add up to 1, in the same order of size as the logits.

Cross-entropy then scores those probabilities against the true label, written as a list y with a 1 for the correct class and 0 for the rest (a one-hot label):
eq. 8 · cross-entropy C = −∑i yi log(ŷi)

In words: only the correct class's term survives, because every other yi is 0. The loss is minus the log of the probability the model gave the correct class. Probability near 1 gives a loss near 0; probability near 0 sends the loss toward infinity.

If the true class is class 0 (so y = [1, 0]) and the model predicts ŷ = [0.9, 0.1], the loss is −log(0.9) ≈ 0.105. If the model says ŷ = [0.1, 0.9], the loss is −log(0.1) ≈ 2.303. Confident wrong answers cost a lot and confident right ones cost almost nothing, which pushes training hardest on the examples it gets most wrong.
Softmax + cross-entropy

When you pair softmax with cross-entropy and work through the calculus, almost everything cancels. You're left with one line: the gradient of the loss with respect to the logits is (ŷ − y), predicted minus actual. It stays large exactly when the model is confidently wrong, whereas sigmoid + MSE multiplies the error by a σ′ that is nearly zero there. That, plus the probabilistic reading of the output, is why softmax + cross-entropy is the standard pairing for classifiers.
Check Your Understanding Q1 of 2

True class is index 0. Model A predicts [0.6, 0.4]; Model B predicts [0.95, 0.05]. Which has lower cross-entropy loss on this example, and by how much?
Model A, its prediction is closer to 0.5, which is safer.
Model B, by a factor of about 10: −log(0.6) ≈ 0.51 vs. −log(0.95) ≈ 0.051.
They’re equal, both are correct.
0/2 correct
