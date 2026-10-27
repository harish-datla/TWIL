# Minibatches trade noise for speed

A training set can hold millions of examples, and the exact gradient is an average over all of them. Paying for a full pass over the data to take one small step is wasteful. Stochastic gradient descent (SGD) instead estimates the gradient from a small random sample of examples, called a minibatch, the way a poll estimates an election from a thousand voters.

Averaged over all the batches you could draw, the estimate equals the full gradient, which makes it an unbiased estimate. So the steps point the right way on average. Each individual step is noisy around that average.[3]

Worked example

Four examples whose gradients for one weight are 2, 4, 6 and 8. The full gradient is their average, 5.

A batch of 1 picks one example at random. The estimate is 2, 4, 6 or 8: it averages 5, but its root-mean-square miss is √5 ≈ 2.24.

A batch of 4, drawn at random with repeats allowed, misses by 2.24 / √4 ≈ 1.12: half the noise for four times the work.

That's the general rule for averages from the probability page: averaging B independent samples divides the spread by √B, so the noise of a B-example gradient falls like 1/√B. Halving the noise costs four times the examples per step. Batch sizes used in practice, often 32 to 256, balance that against memory and hardware throughput.[1]

The noise doesn't fade as training goes on. With a constant learning rate, SGD ends up jittering around the minimum instead of settling on it, which is why the standard convergence guarantees for convex losses shrink the learning rate over time.[3]
Run three batch sizes on the same data and measure the noise
Highlight batch size
Seed
1
Step
Play
Reset
0 updates at η = 0.05, seed 1. Full-data loss now: batch 1 → 32.1760, batch 4 → 32.1760, full batch (24) → 32.1760; optimum 0.3551. Gradient noise (RMS distance from the full gradient at a fixed point, 300 draws): batch 1 13.063, batch 4 6.874, batch 24 0.000; the ratio 1 : 4 is 1.90, and √4 = 2.

Press Step or Play to run SGD three times from the same start, with batches of 1, 4 and 24 (the whole dataset). Each Step applies five updates at η = 0.05, and the left panel plots each run's loss on all 24 points; change the seed to draw different batches. The right panel measures the noise directly: at a fixed point it draws 300 random batches of each size and computes the root-mean-square distance between the batch gradient and the full gradient. Batch 1 comes out at 13.1 and batch 4 at 6.9, a ratio of 1.90, close to √4 = 2. The full batch has zero noise and the smoothest curve, and it costs 24 example-gradients per update where batch 1 costs one.
Check Your Understanding Q1 of 2

Gradient descent on a bowl-shaped quadratic loss diverges at η = 0.1. What does that tell you about the loss?
Its steepest curvature, the largest Hessian eigenvalue, is above 20.
It has no minimum.
The gradient was computed with the wrong sign.
0/2 correct
