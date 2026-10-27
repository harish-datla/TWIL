# Add noise with a recipe you control

Say you want a program that generates new images that look like the ones in a training set. There's no formula for "a realistic photo" to aim at. A diffusion model gets around that. Training takes real examples and adds random noise to them in a fixed, known way. The training code drew that noise itself, so it knows exactly what the noise was and can ask a network to predict it, with no human labeling. To generate, the model starts from pure noise and repeatedly removes some of the noise it predicts.[1]

The recipe blends the clean value with noise drawn from a Gaussian, the bell-curve distribution, using two weights: one shrinks the signal and one scales the noise. Their squares add up to 1, which is why the recipe is called variance-preserving: if the clean data has variance 1, the noisy data does too. One number, ᾱ ("alpha-bar"), sets the noise level. It's the fraction of the signal's power that's left: ᾱ = 1 is clean, and ᾱ near 0 is almost pure noise. Training can jump straight to any noise level in one line, without adding noise step by step.[1]

Worked example

Clean value x₀ = 2, noise draw ε = −0.8, noise level ᾱ = 0.4.

Signal weight √0.4 ≈ 0.632. Noise weight √0.6 ≈ 0.775.

Noisy value x_t = 0.632 × 2 + 0.775 × (−0.8) ≈ 1.265 − 0.620 = 0.645.

These are the defaults of the last widget on this page, which reads x_t = 0.64523.

The full recipe builds ᾱ from a per-step schedule, then gives the one-line jump used in the example.
α_t = 1 − β_t; ᾱ_t = product of α₁ through α_t
x_t = √ᾱ_t × x₀ + √(1 − ᾱ_t) × ε

In words: step t keeps a fraction α_t of the signal's power and replaces the rest, β_t, with noise. Multiply those kept fractions over every step so far and you get ᾱ_t, the signal power left after t steps. Any noisy version is then √ᾱ_t times the clean value plus √(1 − ᾱ_t) times a fresh draw ε from a standard bell curve (mean 0, variance 1).

For one fixed clean value x₀, the noisy x_t is spread around √ᾱ_t x₀ with variance 1 − ᾱ_t, where variance is the average squared distance from the mean. That's only the noise's spread. Across a whole dataset the clean values vary too, so the total variance of x_t is ᾱ_t times the data's variance, plus 1 − ᾱ_t.

Even a small 256×256 color image is 196,608 numbers, far too many for exact answers. So this page uses a dataset with one number per example. Half the values come from a bell curve centered at −2 and half from one centered at +2, each with variance 0.16 (a standard deviation of 0.4). That's a mixture of two Gaussians: two clearly separate clusters, standing in for two kinds of picture. At noise level ᾱ, each cluster's center shrinks to ±2√ᾱ and its variance becomes 0.16ᾱ + 1 − ᾱ.
Follow a mixture as noise increases
Noise level
55
Reset
ᾱ=0.4218; signal coefficient 0.6494; noise coefficient 0.7604; noisy component variance 0.6457. Same 96 clean/noise pairs at each setting.

You're looking at 96 examples from the two-cluster dataset after adding noise (left) and the exact density of the noisy data (right). Drag the noise level: at 0 the clusters sit at ±2, and as noise rises they slide toward 0 and blur into one bell curve. At the default of 55, ᾱ = 0.42, so the cluster centers have moved to about ±1.30 and each has variance 0.65. Height only spreads the dots out. Each example keeps its own noise draw as you drag so you can follow it; that's for comparison, and it isn't how the forward process draws noise step by step.

How fast ᾱ falls from 1 to 0 across the steps is the noise schedule, and it decides how much training the model gets at each noise level. Nichol and Dhariwal proposed a cosine-shaped schedule with a small offset.[2] The widget's slider uses a plain curve, ᾱ = cos²(slider/100 × 90°), which covers the full range but isn't their exact schedule.
