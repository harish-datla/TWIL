# Generalization and early stopping

You train a model for 3,000 epochs (passes over the training data), and the training loss falls the whole way. Which checkpoint should you keep? Not necessarily the last one. Comparing training error with validation error tells you where the model stopped learning things that carry over to new data.

Comparing the two errors shows two failure patterns. When both stay high, the model can't even fit its training data: that's underfitting. When training error is far below validation error, the model has learned quirks of its training sample that don't recur in new data: that's overfitting.[1] The linear models page shows overfitting growing with model complexity. It also grows with training time. A flexible model tends to fit the clean structure of its data first and the noise later, so the validation error falls, bottoms out and rises again while the training error keeps falling.[4]

Early stopping uses that curve directly: check the validation error after each epoch, stop once it hasn't improved by more than a small margin for a set number of epochs, and keep the weights from the best epoch.[4]

Worked example

Validation loss after epochs 1 to 7: 0.50, 0.41, 0.37, 0.36, 0.37, 0.38, 0.39. Stop after 3 epochs without improvement.

The best is 0.36 at epoch 4. Epochs 5, 6 and 7 don't beat it, so training stops after epoch 7 and you keep the epoch-4 weights.

The validation set has now chosen the stopping epoch, so it's been used for a decision. The final number has to come from the test set.
Train past the point of best validation error
Seed
1
Step
Play
Reset
Degree-9 polynomial, 16 training points, 60 validation points, gradient descent at η = 0.5 for 0 epochs (seed 1). Now: training MSE 1.0819, validation MSE 1.1841. Lowest validation error so far: 1.1841 at epoch 0. Noise variance 0.1225 is the floor.

Left: training error (purple) and validation error (green) on a log scale, with the best validation point in yellow. Right: the fitted curve against the training points (green) and validation points (grey). Press Play, or Step for 50 epochs at a time, and try other seeds. The model is a degree-9 polynomial fit to 16 training points by gradient descent. With seed 1 the validation error bottoms out near 0.22 around epoch 90 and climbs past 1.7 by epoch 3,000, while the training error keeps falling toward 0.09; the right panel shows the fit growing wiggles between the training points. The yellow dot is where early stopping would have stopped. Gradient descent from zero weights fits the low-order shape first, which is why the curve has a minimum at all.

The usual way to describe this is the bias–variance trade-off. A model's expected squared error splits into three parts. Bias is the error from the model being too rigid to fit the true pattern. Variance is the error from the model fitting the particular sample it happened to get, so it would come out different on another sample. The noise in the targets is a floor no model gets below.[5] A rigid model has high bias; a flexible model fit to a small sample has high variance. Early stopping, weight decay and more data all reduce the variance part.
