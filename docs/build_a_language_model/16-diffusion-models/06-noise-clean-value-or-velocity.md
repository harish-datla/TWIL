# Noise, clean value or velocity: what the model predicts

A diffusion model's output doesn't have to be the noise. Some models predict the clean value x₀ instead, and some predict a blend of the two called the velocity, v.[6] All three carry the same information when they're exact, but the sampler has to know which one it's getting. Feed a velocity output to a sampler that expects noise and it produces wrong samples without any error.

With s = √ᾱ as the signal weight and σ = √(1 − ᾱ) as the noise weight, the velocity is v = sε − σx₀. The name is a convention; it doesn't describe anything moving.

Worked example

Same values as before: ᾱ = 0.4, x₀ = 2, ε = −0.8, so s ≈ 0.632, σ ≈ 0.775 and x_t ≈ 0.645.

Velocity: v = 0.632 × (−0.8) − 0.775 × 2 ≈ −0.506 − 1.549 = −2.055.

Back to the clean value: s × x_t − σ × v ≈ 0.408 + 1.592 = 2.000. Back to the noise: σ × x_t + s × v ≈ 0.500 − 1.300 = −0.800.

The three conversions, all using the same two weights:
x_t = s × x₀ + σ × ε; v = s × ε − σ × x₀
x₀ = s × x_t − σ × v
ε = σ × x_t + s × v

In words: the noisy value and the velocity are two different blends of the same clean value and noise. Because s² + σ² = 1, the pair (x_t, v) is the pair (x₀, ε) rotated, and rotating back recovers either one exactly. Getting x₀ from a noise prediction instead means dividing by s.

With exact values all three routes agree. With a model's imperfect predictions they don't. Recovering x₀ from a noise prediction multiplies the noise error by −σ/s, which grows without limit as ᾱ approaches 0: at ᾱ = 0.01, a noise error of 0.1 becomes a clean-value error of about 1. Salimans and Ho discuss the prediction target and the loss weighting together for this reason; the targets converting into each other doesn't make the training objectives equivalent.[6]
Reconstruct from exact target representations
Signal power ᾱ
0.4
Clean value x₀
2
Noise ε
-0.8
Reset
x_t=0.64523, v=-2.05516. Recovered x₀ from ε=2.00000, from v=2.00000; recovered ε=-0.80000. These are exact supplied targets, not prediction-quality measurements.

You're looking at the clean value, the noise, the noisy value and the velocity they produce (left), and the clean value recovered from the noise and from the velocity (right). Drag ᾱ, the clean value or the noise. At the defaults (ᾱ = 0.4, x₀ = 2, ε = −0.8) the readout shows x_t = 0.64523 and v = −2.05516, and both routes recover exactly 2. This checks the algebra with exact values. It says nothing about how well a trained model predicts any of them.

Store the prediction type with the model weights and the schedule. Converting needs the current s and σ, and matching array shapes won't catch a mix-up.
