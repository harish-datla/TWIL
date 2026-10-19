# Normalization layers

A good starting scale only fixes the start of training. Once the weights begin to change, so does the size of each layer's output. Rather than hope it stays usable for the whole run, a normalization layer forces it: it takes each example's list of activations and rescales it to a standard size, at every step and after training too.

Layer normalization (LayerNorm) works on one example at a time. It subtracts the mean of that example's features and divides by their standard deviation, so the list ends up with mean 0 and variance 1. Then it multiplies each feature by a learned gain and adds a learned bias, so the model can still pick a different scale where that helps.[3] The original transformer and GPT-2 use it; the transformer page covers where it sits in the block.

Worked example

One example's activations: x = (1, 3, 5, 7).

LayerNorm: the mean is 4 and the variance is (9 + 1 + 1 + 9)/4 = 5, so the standard deviation is √5 ≈ 2.24. With gain 1 and bias 0 the output is (−1.34, −0.45, 0.45, 1.34).

RMSNorm skips the mean: the root mean square is √((1 + 9 + 25 + 49)/4) = √21 ≈ 4.58, so the output is (0.22, 0.65, 1.09, 1.53).

Multiply x by 1,000, as an exploding layer might, and both outputs come out the same.

RMSNorm is the cheaper variant Llama uses. It drops the mean subtraction and the bias and only divides by the root mean square, the square root of the average squared value. This is the whole computation, for each feature i:
yi = gi · xi / √(mean(x²) + ε)

In words: divide every feature by the vector's typical size, then let a learned gain per feature scale it back up. The ε is a tiny constant that keeps the division finite when every feature is near zero. Llama's reference code is exactly this formula; its RMSNorm class defaults to ε = 10⁻⁶, and the model is built with ε = 10⁻⁵.[4]

Both LayerNorm and RMSNorm work on one example at a time, so they compute the same function during training and when the model serves requests. Batch normalization, an older method, differs here: during training it normalizes with averages taken across the whole batch.[5]

Go back to the signal widget and switch any exploding or vanishing setting to LayerNorm or RMSNorm: the variance sits at 1 after every layer, whatever the weight scale. Normalization doesn't make initialization irrelevant, since the size of the gradients and of the values going into each normalization still depend on the weights. What it removes is the exponential dependence on depth that makes a forty-layer stack so sensitive to its starting scale.
Check Your Understanding Q1 of 2

A 50-layer ReLU network is initialized with weights of variance 1/n_in. What happens to the forward signal?
It is preserved: 1/n_in is the variance-preserving choice.
It shrinks by roughly a factor of two per layer, to around 10⁻¹⁵ overall.
It explodes, because ReLU is unbounded.
0/2 correct
