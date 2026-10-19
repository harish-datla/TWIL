# Keeping the signal alive through depth

You build a forty-layer network, start training, and the loss doesn't move. Or it turns into NaN within a few steps. A common cause is the size of the numbers flowing through the network. Each layer multiplies its input by a matrix of weights and passes the result through an activation function, so the typical size of the signal gets multiplied by some factor at every layer. Forty layers multiply forty of those factors together.

Worked example

Each layer shrinks the signal to 0.7 of its size. After forty layers: 0.7⁴⁰ ≈ 6×10⁻⁷, about a millionth of the input.

Each layer grows it by 1.4 instead. After forty layers: 1.4⁴⁰ ≈ 7×10⁵, nearly a million times the input.

Training sends gradients backward through the same layers, multiplying by the same weights, so they tend to shrink or grow in the same way.[1] Gradients that shrink to almost nothing leave the early layers unable to learn; gradients that blow up wreck the weights in a single update. These are vanishing and exploding gradients, and the first lever against them is the scale of the random numbers the weights start with.

To choose that scale, measure the signal by its variance, which for numbers centred on zero is their average squared size. A layer's output variance is easy to predict when its weights are drawn at random around zero.

Worked example

A layer has 64 inputs, each with variance 1. Its weights are drawn with standard deviation 1/√64 = 0.125, so each weight has variance 1/64.

Each output adds up 64 terms of the form weight × input, and each term has variance 1/64 × 1. Independent random terms add their variances, so the output variance is 64 × 1/64 × 1 = 1. The signal keeps its size.

The same reasoning gives the general rule, which tells you the weight variance that keeps a layer's output the same size as its input:
Var[oi] = nin σ² γ²

In words: every input adds its own small random contribution, so the output's spread grows with the number of inputs and with the size of the weights. Choose a weight variance of σ² = 1/nin and a layer with no activation keeps the signal's size.[1]

An activation function changes the bookkeeping. ReLU sets every negative number to zero and passes positive ones through unchanged. When its inputs are spread evenly around zero, that zeroes half of them and halves their average squared size. At the 1/√nin scale, each ReLU layer therefore halves the signal, and forty layers leave 2⁻⁴⁰ ≈ 10⁻¹². Kaiming initialization (also called He initialization) pays that back by doubling the weight variance to 2/nin, a standard deviation of √2/√nin.[2]

Worked example

Width 64 with Kaiming weights: standard deviation √2/8 ≈ 0.177, variance 2/64 = 1/32.

The layer's outputs, before the ReLU, have variance 1. ReLU zeroes the negative half, leaving an average squared size of 0.5. The next layer's output variance is 64 × 1/32 × 0.5 = 1, so the signal holds steady.
Why some libraries use 2/(n_in + n_out) instead

Push a signal through forty random layers
Weight scale
Activation
Normalization
Seed
1
Reset
40 layers of width 64, weights N(0, (1.00/√64)²), relu, no normalization, seed 1. Mean squared activation: layer 0 → 1.004, layer 10 → 6.15e-3, layer 40 → 1.63e-12. Geometric mean factor per layer 0.507.

The line is the mean squared activation after each of 40 layers, on a log scale; the grey line marks 1. For a signal centered on zero that's its variance; after a ReLU it's the average squared size the next layer sees. Try each weight scale with ReLU, then change the activation and the normalization. Every layer is 64 units wide with its own seeded random weights, and the values are measured from the actual activations. The default, ReLU with 1/√n weights, loses about half of it per layer and reaches 10⁻¹² by layer 40. The Kaiming √2 gain keeps it between about 0.03 and 7 at every seed: it drifts at random instead of decaying. 2/√n doubles it per layer, to about 10¹² at layer 40. With no activation, the 1/√n scale wanders but stays within a few orders of magnitude, because the average factor per layer is 1 and only the random fluctuations add up. tanh at the 1/√n scale shrinks steadily, because tanh always returns a smaller number than it's given (|tanh x| < |x|); PyTorch's recommended tanh gain is 5/3.[2] Either normalization pins the variance at 1 whatever the scale; section 2 explains how.

Weight scale isn't the only lever. On the way back, each layer also multiplies the gradient by the slope of its activation function. The sigmoid's slope is at most 1/4 and close to zero for large inputs, so a stack of sigmoids shrinks gradients at every layer even with well-scaled weights. ReLU's slope is exactly 1 wherever its input is positive, which is a large part of why it replaced the sigmoid in deep networks.[1]

The starting weights also have to differ from one another, which is why they're drawn at random. If every unit in a layer starts with the same weights, they all compute the same output, get the same gradient and stay identical forever, so the layer behaves like a single unit.[1]
