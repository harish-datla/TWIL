# Generate by stepping toward less noise

To generate, start from pure noise at a very high noise level. Ask the model what noise it sees, remove part of it, and land at a slightly lower noise level. Repeat until the noise is gone. Some samplers add a little fresh randomness at each step. The DDIM update used here, with its randomness setting η at 0, adds none.[4]

Each step does two things. First it estimates the clean value by running the noise recipe backward with the predicted noise. Then it rebuilds a noisy value at the next, lower noise level from that estimate and the same predicted noise.

Worked example

Current value x_t = 0.645 at ᾱ = 0.4, stepping to ᾱ_next = 0.7. The exact predictor for the toy mixture says ε̂ ≈ −0.490.

Clean estimate: x̂₀ = (0.645 − 0.775 × (−0.490)) / 0.632 = 1.025 / 0.632 ≈ 1.621.

Next value: √0.7 × 1.621 + √0.3 × (−0.490) ≈ 1.356 − 0.269 = 1.087.

With the true noise, −0.8, the clean estimate would be exactly 2. The code in section 07 checks that case.

Written out, the two parts of the step are:
x̂₀ = (x_t − √(1 − ᾱ_t) × ε̂) / √ᾱ_t
x_next = √ᾱ_next × x̂₀ + √(1 − ᾱ_next) × ε̂

In words: estimate the clean value by subtracting the predicted noise at its current weight and undoing the signal's shrink. Then blend that estimate with the same predicted noise, using the next level's weights. ᾱ_next is larger than ᾱ_t, so each step carries more signal and less noise. Both parts come from one model prediction.

With η = 0 the only randomness is the starting noise. The same starting draw always gives the same output, and different draws give different outputs. A stochastic sampler, such as the original DDPM one, draws fresh noise at every intermediate step; DDPM's final step adds none.[1]
Run deterministic reverse trajectories
Reverse steps
24
Step
Play
Reset
0/24 deterministic reverse steps; 64 initial Gaussian draws (seed 122). Exact mixture conditional-mean noise predictor, not a trained neural model. ᾱ at start is 10⁻⁴, so Gaussian initialization approximates its noisy marginal.

You're looking at 64 samples as they move (left) and the paths of six of them (right). Press Step to run one reverse update for every sample, or Play to run them all; changing the number of steps restarts. The samples start as random draws from a bell curve centered at 0 and, after the default 24 steps, end in two clusters near −2 and +2. The noise predictor is the exact best guess for the known mixture, not the line trained above. Fewer steps make each update coarser: at 8 steps the clusters end slightly closer to 0.

The demo starts at ᾱ = 10⁻⁴, not exactly 0, so a trace of the data remains, and a standard bell curve is a close stand-in for the noisy data there. Even with the exact predictor, the finite number of steps and that approximate start keep the output from being an exact draw from the mixture. When samples look wrong, both the predictor and the sampler are suspects.

In an image model, x_t is a whole array of pixels, or of latent values from a compressed version of the image, and the network predicts a noise array of the same shape. The update above runs on each number separately, but the network looks at all of them together to decide what noise it sees. The one-number toy isolates the sampling arithmetic; it doesn't generate anything like an image.
