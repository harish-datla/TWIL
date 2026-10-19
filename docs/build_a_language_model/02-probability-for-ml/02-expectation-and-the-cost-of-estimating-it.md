# Expectation and the cost of estimating it

You can't compute a model's loss over every sentence that could ever be written. You compute it on a batch or a test set and treat that as the answer. This section says what that number estimates and how far off it can be, which tells you how many examples you need before a difference between two models is real.

The number being estimated is an expectation: the average value of something, weighted by how likely each value is. Multiply each possible value by its probability and add the results. A model's true loss is an expectation: the loss on each possible input, weighted by how often that input occurs.[1]

Worked example

A fair die: (1 + 2 + 3 + 4 + 5 + 6) × 1/6 = 3.5.

A model whose loss is 0.1 on the 70% of inputs that are easy and 2.0 on the 30% that are hard: 0.7 × 0.1 + 0.3 × 2.0 = 0.67.

The formula below writes that weighted sum, then the approximation you can actually compute: draw n samples and take their plain average.
E[f] = Σx P(x) f(x) ≈ (1/n) Σi f(xi)

In words: the expectation adds up every possible value times its probability. In practice you replace it with the plain average of n random samples. An estimate made by averaging random samples is called a Monte Carlo estimate.

A Monte Carlo estimate is noisy. For a plain average, the only way to shrink the noise is to draw more samples, and that works slowly.[1] How noisy depends on how spread out a single sample is. That spread is measured by the variance, σ², the average squared distance of a sample from the mean. Its square root, σ, is the standard deviation.

Averaging n independent samples divides the variance by n, so the average's typical error, its standard error, is σ/√n.[5] Four times the samples only halves the error.

Worked example

A model gets 80% accuracy on 100 test examples. Each example scores 1 (right) or 0 (wrong), so σ = √(0.8 × 0.2) = 0.4.

Standard error: 0.4 / √100 = 0.04, four percentage points. A rival scoring 81% on those 100 examples got one more answer right, well inside that noise.

Pinning the accuracy down to about one point needs 0.4 / √n = 0.01, so n = 40² = 1,600 examples.

The same 1/√n rate is why a gradient computed on a small minibatch of training examples is noisy.
Average samples and watch the error shrink like 1/√n
Distribution
Seed
1
Step
Play
Reset
10 samples (seed 1). Running mean 4.1000; true mean 3.5000; absolute error 0.6000. Standard error σ/√n = 1.708/√10 = 0.5401. Error inside one standard error: no (expected about two thirds of the time).

The purple line is the running average of samples from the chosen distribution, and the grey band is the true mean ± σ/√n. Press Step to add ten samples, or Play to keep adding them. The band narrows with the square root of the count, so the last 500 samples tighten it far less than the first 50 did. A different seed gives a different path inside the same band. For large n the average's error follows a bell curve (the central limit theorem), which puts the estimate inside the band roughly two thirds of the time.
