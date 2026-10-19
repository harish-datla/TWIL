# A line, a loss, and what the loss assumes

Say you want to predict an apartment's monthly rent from its floor area, its number of rooms and its distance to the nearest station. Those inputs are the features. The simplest useful guess multiplies each feature by a weight, adds up the results, and adds one more constant called the bias, which sets the prediction when every feature is zero. That's linear regression.[1]

In symbols the prediction is ŷ = w · x + b: the dot product of the weight vector w with the feature vector x, plus the bias. A dot product multiplies matching entries and adds them up; Linear Algebra covers it. Learning means finding the w and b whose predictions best match examples where you already know the answer.

"Best" needs a number. For each example, the residual is the prediction minus the true answer: the vertical gap between a data point and the line. Square each residual so that misses in either direction count against the model, then average them over the examples. That average is the mean squared error (MSE). It's the model's loss: lower is better, and zero means every prediction is exact.

Worked example

Three examples (x, y): (1, 2), (2, 3) and (3, 5).

Try the line ŷ = 1.5x. Predictions 1.5, 3, 4.5. Residuals −0.5, 0, −0.5. MSE = (0.25 + 0 + 0.25) / 3 ≈ 0.167.

Add a bias of 1/3: ŷ = 1.5x + 0.333. Residuals −0.167, 0.333, −0.167. MSE = (0.028 + 0.111 + 0.028) / 3 ≈ 0.056. No other line does better on these three points.

The formula below computes that score for any choice of weights and bias. It's the number the training loop will try to push down.
L(w, b) = (1/n) Σi (wᵀxi + b − yi)²

In words: for every example, take the prediction minus the true answer, square it, and average those squares over all n examples.

Squaring punishes big misses far more than small ones: a miss of 2 costs four times as much as a miss of 1. The fit works hard to avoid large errors, which also means a single wild outlier can drag the whole line toward itself.[1]

The choice of squared error isn't arbitrary. Suppose each true answer is the line's prediction plus random noise with a bell-curve (Gaussian) shape. Then the weights that make your observed data most probable are exactly the weights with the lowest MSE.[1] Picking parameters that make the data most probable is called maximum likelihood, and the probability page covers it. The same idea produces the loss for yes-or-no answers later on this page.
Why Gaussian noise leads to squared error

Fit a line by hand, compare with the least-squares optimum
Slope w
0.5
Bias b
-1
Reset
ŷ = 0.5·x − 1: mean squared error 4.7768 over 24 points. Closed-form least squares: w* = 1.7224, b* = 0.0784, MSE* = 0.3551. Residuals are the vertical gaps; the loss is the mean of their squares.

Drag the slope w and the bias b to move the purple line through the 24 green points; the pink segments are the residuals. The right panel shows the MSE as w varies with b held where you set it. That curve is a parabola, because the loss is quadratic in the weights, and its lowest point shifts as you change b. The points were generated as y = 1.5x + 0.5 plus Gaussian noise with standard deviation 0.6, so no line reaches zero: the best one, shown in the readout, has MSE 0.355, close to the noise variance of 0.36.

Real datasets have many examples and many features. Code stacks the examples as the rows of one table, the design matrix X, with one column per feature. All n predictions then come out of a single matrix–vector product, Xw, and the bias can ride along as an extra column of ones.[1]

For linear regression you don't need to search for the best weights at all. Plotted against the weights, the MSE is a bowl. At the bottom of a bowl the slope is zero in every direction. Writing that condition out gives a set of linear equations, the normal equations XᵀX w = Xᵀy, whose solution w* = (XᵀX)⁻¹Xᵀy is the best fit. It's unique as long as no feature column can be built from the other columns.[1][6] With a single feature the answer reduces to slope = covariance(x, y) / variance(x), which gives exactly the 1.5 and 1/3 of the worked example.

Solving the normal equations directly gets fragile when two features carry nearly the same information, such as floor area and number of rooms. XᵀX is then close to a matrix that has no inverse, and small changes in the data can push the fitted weights to large values.[6] Adding a penalty on large weights, the weight decay covered below, keeps the equations well behaved.[6]
