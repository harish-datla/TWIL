# Gradient descent and the learning rate

Most models have no formula for their best weights. A neural network with millions of weights has no normal equations to solve. What you can compute is the gradient: for each weight, how fast the loss rises if you nudge that weight up a little. Move every weight a small step the opposite way and the loss goes down. Repeat that thousands of times and you have gradient descent. The size of each step is set by one number, the learning rate, written η (the Greek letter eta).[1]

Worked example

One weight, loss L(w) = (w − 3)², lowest at w = 3. Its slope (derivative) is 2(w − 3).

Start at w = 0 with η = 0.1. Slope −6, so w becomes 0 − 0.1 × (−6) = 0.6. Slope −4.8, so w becomes 1.08, then 1.464. Each step closes 20% of the remaining gap to 3.

The update below does the same thing for every weight at once. It's the simplest form of the step a training loop takes; later pages refine it.
w ← w − η ∇L(w)

In words: the new weights are the old weights minus the learning rate times the gradient. The gradient points uphill, so subtracting it moves downhill.

For the MSE the gradient has a short form, ∇L = (2/n) Xᵀ(Xw − y). Read it right to left: compute every residual (Xw − y), multiply each one by its example's features, average, and double. A weight whose feature lines up with large residuals gets a large push.[6] The loss for yes-or-no answers, further down, has a gradient of the same shape.

The learning rate can't be as large as you like. Rerun the worked example with η = 1.1: w jumps from 0 to 6.6, past the minimum, then to −1.32, then to 8.18. The gap to 3 grows by a factor of 1.2 every step, flipping sides each time, and the loss climbs without limit.

The limit comes from how sharply the loss bends around its minimum, its curvature (the second derivative). For a bowl with curvature λ, each step multiplies the distance to the minimum by (1 − ηλ). That distance shrinks only while |1 − ηλ| < 1.[2] In the worked example the curvature is 2, so at η = 0.1 the factor is 0.8 and at η = 1.1 it's −1.2.
η < 2 / λmax

In words: the step size must stay below 2 divided by the curvature of the loss's steepest direction. Past that, every step overshoots the minimum by more than the step before.

With many weights, the bowl bends by different amounts in different directions. The curvatures along the bowl's main axes are the eigenvalues of the loss's Hessian, the matrix of its second derivatives. (An eigenvector is a direction a matrix only stretches, and its eigenvalue is the stretch factor; Linear Algebra covers them.) One learning rate serves every direction, so it has to suit the steepest one, and the flattest direction then crawls. The ratio of the steepest curvature to the flattest is the condition number.

The data sets that ratio, because a feature's curvature grows with the square of its scale. Give a model one length in millimetres and another in kilometres and the curvatures can differ by about 10¹², a bowl whose axes differ a million to one.[2] That's why inputs are usually standardized to zero mean and unit variance before fitting.[8] Diagonal preconditioning attacks the same problem from the optimizer's side: it scales each weight's step by that weight's own curvature, which amounts to a separate learning rate per weight.[2]
Descend the loss bowl; stretch it by rescaling the feature
Learning rate η
0.2
Feature scale
Step
Play
Reset
Feature scale ×1: Hessian eigenvalues 3.533 and 1.761, condition number 2.0. Stable learning rate η < 2/λ_max = 0.566; η = 0.2 converges. After 0 steps: (b, w) = (-2.500, -2.500), loss 4.295e+1; optimum (0.078, 1.722).

The left panel shades the loss over every (w, b), dark where it's low; press Play to watch descent start at (−2.5, −2.5) and head for the green optimum, while the right panel tracks the loss per step. With the feature as is, the bowl's curvatures (Hessian eigenvalues) are 3.53 and 1.76, the condition number is 2.0, and any η below 0.566 converges. Switch to "x multiplied by 4" and the bowl becomes a narrow trench: curvatures 52.8 and 1.88, condition number 28, and a ceiling of 0.038, so the default η = 0.2 diverges within a few steps. Lower η to 0.03 and it converges, but slowly along the flat direction: after 60 steps w is within 0.003 of its optimum while b, which lies mostly along that flat direction, is still 0.07 short.
