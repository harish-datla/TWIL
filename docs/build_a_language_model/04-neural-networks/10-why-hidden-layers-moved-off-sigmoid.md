# Why hidden layers moved off sigmoid

Train a ten-layer network with sigmoid in every hidden layer and the last layers learn while the first ones barely change. BP2 explains why: every layer the error passes back through multiplies it by that layer's activation slope, σ′(z). The sigmoid's slope is at most 0.25, at z = 0, and falls toward zero as z moves away from zero in either direction (0.105 at z = 2, 0.018 at z = 4).

Worked example

Ten sigmoid layers, every neuron in the best case z = 0, so every slope is 0.25.

Slope factors alone: 0.2510 ≈ 9.5 × 10−7, about a millionth.

With every z at 2 instead, each factor is 0.105, and ten of them give about 1.6 × 10−10.

Unless the weights happen to be large enough to compensate, the gradient reaching the first layer is around a millionth of the gradient at the output, and those early weights barely move. This is the vanishing gradient problem[5], and through the 1990s and 2000s it made deep sigmoid and tanh networks very hard to train.

Several fixes arrived around 2006 to 2015 (layer-wise pretraining, better initialization, normalization layers, residual connections). The simplest, popularized around 2010 to 2011, is to replace sigmoid in the hidden layers with the rectified linear unit (ReLU), which passes positive inputs through unchanged and turns negative ones into zero. ReLU(3) = 3 and ReLU(−2) = 0:
eq. 10 · ReLU ReLU(z) = max(0, z)

In words: if z is positive, output z; if it's zero or negative, output 0. In code it's one line: return z > 0 ? z : 0.

The derivative is 1 for positive inputs and 0 for negative ones, so the activation contributes no shrinking factor: the gradient passes through unchanged for any neuron that's "on." Combined with better initialization, ReLU made much deeper networks trainable with plain gradient descent.

ReLU has a known failure mode: a neuron whose pre-activation is negative for every training example produces zero output and zero gradient, so it stops learning for good ("dying ReLU"). Leaky ReLU patches this with a small slope below zero (the widget uses 0.1z; 0.01z is also common). GELU multiplies z by a smooth weight that runs from 0 for very negative z to 1 for large positive z (the probability that a bell-curve random number falls below z), giving a curve with no kink that behaves like ReLU away from zero. GPT-2 uses it in every feed-forward block[6]; many recent large language models use gated variants such as SwiGLU instead. Try them in the activation widget above to see the shapes.
Initialization matters too

If all your weights start at zero, every neuron in a layer computes the same thing and learns the same thing. Gradient descent can never break that symmetry, so networks start from random weights. How big should the random numbers be? Too big and the signal grows layer after layer; too small and it fades away.

The size of random weights is measured by their variance, the average squared distance from their mean; for weights centered on zero, that's just their average square. A neuron with more inputs adds up more random terms, so its weights need to be smaller. Glorot and Bengio showed in 2010[7] that for tanh-like activations, weights with variance 1/nin keep the signal's size roughly constant from layer to layer, where nin is the number of inputs to the layer. ReLU zeroes the negative half of its inputs, which halves the signal's average square at every layer, so for ReLU networks He et al. showed in 2015[8] that you need twice that: variance 2/nin. Get the scale wrong on a deep network and the activations grow or shrink geometrically with depth.
The published Xavier rule

The code below uses He initialization (variance 2/nin) when the activation is ReLU and variance 1/nin otherwise, drawn uniformly from [−limit, limit]. A uniform draw on that range has variance limit²/3, so the code sets limit = √(3 × target variance).

Worked example

A ReLU layer with 8 inputs. Target variance: 2 / 8 = 0.25. Limit: √(3 × 0.25) = √0.75 ≈ 0.87, so each weight starts somewhere between −0.87 and 0.87.

With tanh or sigmoid, the target is 1 / 8 = 0.125 and the limit is √0.375 ≈ 0.61.
Check Your Understanding Q1 of 2

With a sigmoid activation, the slope σ′(z) maxes out at 0.25. In a 12-layer network, counting only the σ′ factors, by at least what factor does the gradient shrink between the output and the input?
About 12× smaller.
About 17 million times smaller: 0.2512 ≈ 6 × 10−8.
The gradient stays the same, since the chain rule cancels out the factors.
0/2 correct
