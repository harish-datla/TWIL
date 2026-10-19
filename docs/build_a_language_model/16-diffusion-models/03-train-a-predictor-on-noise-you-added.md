# Train a predictor on noise you added

Training is the easy part, because the training code makes its own targets. Pick a clean example, pick a noise level, draw noise, build x_t with the recipe, and ask the network ε̂_θ(x_t, t) to predict the noise you drew. The loss is the squared error between that prediction and the real noise; Ho et al.'s simplified objective is exactly this.[1]

Worked example

Take the example from section 01: x₀ = 2, ᾱ = 0.4, ε = −0.8, so x_t ≈ 0.645.

If the network predicts ε̂ = 0, this example's loss is (−0.8 − 0)² = 0.64. A prediction of −0.5 would cost (−0.8 + 0.5)² = 0.09.

The best possible predictor for the toy mixture guesses ε̂ ≈ −0.49 here, because other clean-and-noise pairs also land on 0.645. Its loss on this example is (−0.8 + 0.49)² ≈ 0.096, not 0.

Averaged over many examples, noise draws and noise levels, that gives the training objective.
loss = E[ || ε − ε̂_θ(x_t,t) ||² ]

In words: the average squared gap between the noise you added and the noise the network predicts. The network sees the noisy sample and the noise level t. It never sees the clean example.

No network can recover the exact noise from x_t alone: the clean value is hidden, and different clean-plus-noise pairs produce the same x_t. Under squared error the best possible predictor outputs the expected noise given x_t, the average from section 02. A real network approximates that function from a finite training set.

The next widget trains a tiny model, a straight line ε̂ = w x_t + b, by gradient descent on 64 fixed noisy examples at one noise level. A line can't follow the mixture's curved best-guess function, so the loss falls and then levels off well above zero.
Fit a noise predictor with gradient descent
Signal power ᾱ
0.5
Learning rate
0.05
Step
Play
Reset
Full-batch updates 0; ε̂=w x_t+b, w=0.0000, b=0.0000; training MSE 1.01582. 64 fixed training pairs at one noise level; no held-out metric.

You're looking at the 64 training pairs, each a noisy value against the noise that was added (dots), with the fitted line, and the training loss after each update. Press Step or Play to take gradient steps; changing ᾱ or the learning rate restarts training. At the defaults the loss starts at 1.016 with w = b = 0 and levels off near 0.693 within about 20 steps, at w ≈ 0.336. That's as good as a straight line at one noise level can do.

A real network also takes the noise level as an input, and it needs to: the same noisy value calls for a different noise guess at different levels. Check the loss at each noise level as well as the average, because an average over all levels can hide a range where the model is bad.

Two more cautions. Changing what the network predicts (the noise, the clean value, or the velocity from section 06), or how the loss weights each noise level, changes what training optimizes, even though the predictions convert into each other exactly. And a low denoising loss alone doesn't show that samples look right, cover the data's variety, or hold up under a different sampler.
