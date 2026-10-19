# Attention is one part of a transformer block

Attention moves information between positions, but it only averages values. A transformer block follows it with a position-wise feed-forward network: a small two-layer network that processes each position's vector on its own, using the same weights at every position. Attention gathers; the feed-forward network computes on what was gathered.[2]

Each of those two sublayers is wrapped in a residual addition: the block adds the sublayer's output to its input rather than replacing the input, so the original signal always has a direct path through the stack.

The block also keeps the numbers in a steady range. As vectors pass through dozens of layers, their overall size can drift. Layer normalization rescales each token's vector on its own: subtract the vector's mean, divide by its standard deviation (the square root of its variance), then apply a learned per-feature gain and bias. Each token gets its own statistics. Unlike batch normalization in training, it never borrows statistics from other examples in the batch.[4]

Worked example

One token's vector: (1, 2, 3, 5). Mean 2.75. Deviations −1.75, −0.75, 0.25, 2.25; their squares average to a variance of 2.1875, a standard deviation of 1.479.

Normalized: (−1.18, −0.51, 0.17, 1.52), mean 0 and variance 1. Add 10 to every entry, or multiply every entry by 3, and the normalized vector comes out the same, apart from a tiny effect of the ε below.
normalizedi = (xi − mean(x)) / √(variance(x) + ε)
outputi = gaini × normalizedi + biasi

In words: centre the token's features on zero and scale them to unit variance, then let a learned gain and bias stretch and shift each feature. The variance averages squared deviations over the feature count. Because of the gain and bias, which form a learned affine step, the final output doesn't have to keep mean 0 or variance 1.

The ε makes the scale invariance slightly inexact: multiplying the input by a constant cancels almost completely when the variance is much bigger than ε, but not perfectly. Adding a constant to every feature cancels exactly in the centring step.
Normalize one token's features
Common offset
2
Input scale
1
Learned gain
1
Reset
Raw mean 2.7500, variance 2.1875. Normalized mean 0.000000, variance 0.999995; gained variance 0.999995. Learned gain 1; bias zero; ε=10⁻⁵.

Drag the offset, the scale and the gain. The first panel shows the raw vector, (−1, 0, 1, 3) times the scale plus the offset; at the defaults that's (1, 2, 3, 5), the worked example above. In the second panel purple is the normalized vector (ε = 10⁻⁵) and green applies the learned gain with bias 0. The offset and scale sliders change the raw bars but leave purple unchanged; only the gain changes green. Each panel scales its bars to its own largest value, so compare sizes using the readout.

Where the normalization sits matters. The original transformer uses post-LN, LayerNorm(x + F(x)): add the sublayer's output, then normalize. A pre-LN block uses x + F(LayerNorm(x)): normalize only the sublayer's input, and let the residual path skip the normalization. They compute different functions with different gradient paths, which Xiong et al. study in detail, so weights trained with one can't be loaded into the other.[5]
Check Your Understanding Q1 of 2

What does token-wise LayerNorm average over?
Every example in the minibatch.
Only the keys receiving high attention.
That token's feature coordinates.
0/2 correct
