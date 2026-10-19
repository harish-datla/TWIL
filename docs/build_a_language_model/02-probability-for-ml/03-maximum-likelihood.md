# Maximum likelihood

Training a model means picking its weights. The rule behind most training is maximum likelihood: pick the weights under which the data you actually saw was most probable. A coin shows the whole idea with one parameter.

Say a coin landed heads 9 times in 13 flips, and you want its heads probability θ. Every candidate θ assigns a probability to that exact sequence of flips: θ for each head and 1 − θ for each tail, multiplied together. Read as a function of θ with the data held fixed, that probability is the likelihood. The maximum likelihood estimate is the θ with the highest likelihood.[4]

Worked example

9 heads and 4 tails, so the likelihood is θ⁹(1 − θ)⁴.

At θ = 0.5: 0.5¹³ ≈ 0.000122.

At θ = 9/13 ≈ 0.692: 0.692⁹ × 0.308⁴ ≈ 0.000327, about 2.7 times higher.

No other θ does better. The peak is at θ = 9/13, the fraction anyone would have guessed.

Real datasets often have millions of points, and multiplying millions of probabilities breaks floating point. (1/2)¹¹⁰⁰ is already below the smallest positive double, about 4.9 × 10⁻³²⁴, so the product rounds to zero. The fix is to take the logarithm, which turns the product into a sum, and flip the sign so there's something to minimize. The logarithm always rises when its input rises, so the θ that maximizes the likelihood also minimizes the negative log-likelihood.[4]

The formula writes the likelihood of a whole dataset of independent points, and the loss a training loop actually minimizes.
L(θ) = Πi p(xi | θ),   NLL(θ) = −Σi log p(xi | θ)

In words: the likelihood multiplies the model's probability of each data point. The negative log-likelihood (NLL) adds up minus the log of each one instead. That sum stays a reasonable size, and its minimum sits at the same θ as the likelihood's maximum.
Flip coins, find the likelihood peak
Heads
9
Flips
13
Reset
9 heads in 13 flips. Likelihood L(θ) = θ^9(1−θ)^4; grid argmax θ = 0.69, closed form heads/flips = 0.6923. At θ̂: L = 3.275e-4, log L = -8.024. At θ = 0.5: L = 1.221e-4, log L = -9.011. Negative log-likelihood at θ̂ per flip: 0.6172 nats.

Set the heads and flips. The left panel shows the likelihood of every θ from 0.01 to 0.99, scaled so its peak is 1, and the right panel shows its logarithm. The yellow dot sits at heads/flips, which matches the grid maximum to grid resolution. The log plot has the same peak on a usable scale: with 30 heads in 60 flips the raw likelihood at the peak is 8.7 × 10⁻¹⁹, while its logarithm is −41.6. More flips sharpen the peak, because more data pins θ down more tightly: the 1/√n effect from the previous section.

Most losses in this series are negative log-likelihoods in disguise. Mean squared error is, up to a constant and a scale factor, the NLL of a model that assumes Gaussian noise around its prediction. Cross-entropy, the loss for classifiers and language models, is the NLL of a model that outputs a probability for each class.[3] So picking a loss also picks an assumption about the noise in the data, whether or not anyone writes it down.
Check Your Understanding Q1 of 2

A screening test is 99% accurate on both sick and healthy people, and 1 in 1,000 people are sick. Someone tests positive. Which is nearest to P(sick | positive)?
About 99%.
About 9%.
About 50%, because the test is symmetric.
0/2 correct
