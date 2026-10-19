# Weight decay and overfitting

A spam filter that gets every one of its training emails right can still do badly on tomorrow's mail. It has learned the noise and quirks of its training sample instead of the pattern. Training error far below the error on data the model hasn't seen is overfitting. The number that matters is the error on held-out data, which is why you set aside a validation set the model never trains on.[5]

A linear model can overfit when it has too many features for the amount of data. The widget below shows this with one input x expanded into the features x, x², x³ and so on. A degree-9 polynomial is still a linear model, a weighted sum of 9 features plus a bias, and with only 14 training points it has enough freedom to bend through nearly all of them.

Weight decay fights overfitting by charging the model for large weights. Large weights are what let a high-degree polynomial swing wildly between the points, so penalizing them flattens the fit. The penalty strength λ is a dial: 0 means no penalty, and large values flatten the fit toward a constant. (This λ is a different number from the curvature λ in the learning-rate section; the letter gets reused.) The formula below is the loss the training loop minimizes once the penalty is added.
Ltotal = Ldata + λ/2 · ‖w‖²

In words: the loss is the usual error on the training data plus half of λ times the sum of the squared weights.

The penalty's gradient is λw, so each descent step first shrinks every weight by the factor (1 − ηλ) and then applies the data gradient as usual. That steady shrinking is where the name comes from.[4]

Worked example

A weight of 2, learning rate η = 0.1, penalty λ = 0.5.

Shrink factor 1 − 0.1 × 0.5 = 0.95, so the weight becomes 2 × 0.95 = 1.9 before the data gradient moves it. A weight the data doesn't defend keeps shrinking by 5% per step.

With squared error, the penalized problem still has a closed-form answer, called ridge regression: solve (XᵀX + αI)w = Xᵀy. Adding α to the diagonal of XᵀX also cures the near-duplicate-features problem from the first section.[7][6] With the MSE as the data term, the two penalties match when α = nλ/2. By convention the bias isn't penalized.[4]
Raise the polynomial degree, then add weight decay
Polynomial degree
3
λ
Reset
Degree 3 polynomial, λ = 0: training MSE 0.0328 on 14 points, held-out MSE 0.1282 on 40 points; ‖w‖ (without bias) = 4.67. Noise variance in the data is 0.0625, the expected floor for held-out MSE.

Drag the polynomial degree from 1 to 9 with λ = 0 and compare the two bars on the right: error on the 14 green training points against error on the 40 grey held-out points the fit never saw. Degree 3 gives training MSE 0.033 and held-out 0.128. Degree 9 bends through the training points (training MSE 0.012) and swings wildly between them: held-out MSE 3.3, far worse than a straight line's 0.53. Keep degree 9 and set λ = 10⁻²: pulling the high-order weights toward zero brings the held-out error down to 0.127. Push λ to 100 and the fit flattens toward the mean, underfitting with both errors high. The truth is y = sin(1.5x) + 0.3x plus noise of variance 0.0625, the floor for held-out error. The λ selector sets the ridge coefficient α in (XᵀX + αI)w = Xᵀy, with the bias unpenalized and x rescaled to [−1, 1] before the powers are taken.
Check Your Understanding Q1 of 2

Logistic regression is trained on perfectly separable data with no regularization. What happens to the weights?
They converge to the unique maximum-likelihood solution.
They grow without bound, driving every training probability to 0 or 1.
They oscillate because the loss is not convex.
0/2 correct
