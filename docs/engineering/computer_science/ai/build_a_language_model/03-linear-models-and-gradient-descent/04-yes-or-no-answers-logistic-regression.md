# Yes-or-no answers: logistic regression

A spam filter needs a yes-or-no answer, ideally with a confidence attached. A weighted sum can be any number, from very negative to very positive, so the model computes the same score as before, z = w · x + b, and squashes it into a probability between 0 and 1 with the sigmoid function, σ(z) = 1 / (1 + e⁻ᶻ). A score of 0 becomes 0.5, large positive scores approach 1, and large negative scores approach 0. That's logistic regression, a classifier despite the name.

The loss comes from the same maximum-likelihood idea as squared error, applied to a yes-or-no label. It's the cross-entropy from the probability page: minus the log of the probability the model gave to the correct answer. Confident and right costs almost nothing. Confident and wrong costs a lot.[6]

Worked example

An email scores z = 2, so p = σ(2) ≈ 0.881: the model says 88% spam.

If it is spam (y = 1): loss = −log 0.881 ≈ 0.127, and the error p − y = −0.119, a small push.

If it isn't (y = 0): loss = −log(1 − 0.881) = −log 0.119 ≈ 2.127, and p − y = 0.881, a push about 7 times larger.

The three formulas below compute the probability, the loss for one example, and the gradient the training loop needs.
p = σ(wᵀx + b),   ℓ = −y log p − (1 − y) log(1 − p),   ∇wℓ = (p − y) x

In words: squash the score into a probability, charge minus the log of the probability given to the true label, and push each weight by the prediction error times its input. That last step has exactly the error-times-input shape linear regression had.[6]
Why the gradient comes out so simple

The decision boundary is the line (or, with more features, the flat surface) where z = 0 and the model is exactly 50/50. Points far from it on the correct side have p − y near zero and barely move the weights. Misclassified points and points near the boundary carry most of the gradient.
Train a classifier by gradient descent, then make the classes separable
Learning rate η
0.5
Class separation
Step
Play
Reset
After 0 steps at η = 0.5: w = (0.000, 0.000), bias 0.000, ‖w‖ = 0.000. Mean cross-entropy 0.6931 nats; training accuracy 50.0%. The boundary is where wᵀx + b = 0, so P(y = 1) = 0.5 on it.

Press Play to train on 40 seeded points, 20 per class. The yellow line is the decision boundary, and the right panel tracks the loss (purple) and the length of the weight vector ‖w‖ (green, drawn divided by 10). On the overlapping set no line separates the classes, so the loss settles at a positive value (0.167 after 400 steps) and ‖w‖ levels off near 3: the fit balances the costs of the points it can't get right. Switch to the separable set and the loss keeps falling toward zero while ‖w‖ keeps climbing, slowly and without limit. Scaling w up pushes every probability closer to 0 or 1 without moving the boundary, so maximum likelihood has no finite best answer here, and the weights grow until something else, such as weight decay or a cap on steps, stops them.[6]
