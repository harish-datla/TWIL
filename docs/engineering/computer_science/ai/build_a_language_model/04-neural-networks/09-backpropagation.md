# Backpropagation: every gradient in one pass

Gradient descent needs ∂C/∂w for every weight in the network, potentially millions of them. The naive approach nudges one weight, reruns the whole forward pass, and measures the change in loss, once per weight: millions of forward passes per update. Backpropagation gets all the gradients from one backward sweep instead. It starts at the loss, works back through the layers, and reuses the numbers the forward pass already computed.

The quantity it passes backward is each neuron's error, written δ ("delta"): how much the loss changes per unit change in that neuron's pre-activation z. Once a neuron's error is known, the gradients of the weights and bias feeding into it follow with one multiply each.

Worked example

The smallest network that shows the idea: input x = 2, one hidden ReLU neuron with weight w1 = 0.5, one output with weight w2 = 3 and no activation, no biases, and squared-error loss C = ½(ŷ − y)² with target y = 2.

Forward: h = ReLU(0.5 × 2) = 1, ŷ = 3 × 1 = 3, C = ½(3 − 2)² = 0.5.

Backward: the output's error is ŷ − y = 1. The gradient for w2 is (what flowed into w2) × (error) = h × 1 = 1. Send the error back through w2 and through the ReLU, whose slope is 1 because its input was positive: the hidden error is 3 × 1 × 1 = 3. The gradient for w1 is x × 3 = 6.

Check by nudging: w1 = 0.501 gives h = 1.002, ŷ = 3.006 and C ≈ 0.50602, a change of about 0.006 for a nudge of 0.001, a slope of about 6.

Every number the backward pass used (h, x, w2 and the ReLU's slope) came from the forward pass or the stored weights, so nothing had to be recomputed. Nielsen states the general version as four equations[4]. Two pieces of notation first: ⊙ means multiply two equal-length lists slot by slot (the Hadamard product), and ∇aC is the list of slopes of the loss with respect to each output activation.

BP1 starts the backward pass at the output layer. It gives each output neuron's error from the loss and the output activation's slope:
BP1 · output layer error (general form) δ(L) = ∇aC ⊙ σ′(z(L))

In words: output error = (how the loss responds to each output activation) times (the activation's slope at that output), slot by slot. This is the general form, valid for any output activation and loss; it's what you'd use for sigmoid outputs with mean squared error. In the worked example the output had no activation (slope 1), so the error was just ŷ − y.

The code on this page leaves the output layer as raw logits and uses softmax with cross-entropy. Push those two through the chain rule and the σ′ factor, the softmax Jacobian (the table of how every probability depends on every logit) and most of the algebra cancel, leaving the one-liner from §6:
BP1′ · softmax + cross-entropy shortcut δ(L) = ŷ − y

In words: the output error is the predicted probabilities minus the one-hot label. For probabilities (0.6, 0.4) with class 2 correct, the label is (0, 1) and the error is (0.6, −0.6). There's no σ′ to evaluate and no Jacobian to build; the code in §12 uses exactly this line.

BP2 moves the error back one layer. It's the step that turned an output error of 1 into a hidden error of 3 in the worked example:
BP2 · propagate error backward δ(l) = ((W(l+1))⊤ δ(l+1)) ⊙ σ′(z(l))

In words: error at layer l = (next layer's weights, transposed, times next layer's error), then times this layer's activation slope, slot by slot. The transpose (rows and columns swapped) sends each error back along the same weights it came through. Repeat it layer by layer down to the first hidden layer.

The last two equations turn errors into gradients. For a bias:
BP3 · gradient wrt. bias ∂C∂bj(l) = δj(l)

In words: the bias gradient is the neuron's error itself. A bias adds straight into z, so one unit of bias moves z by exactly one unit.

And for a weight:
BP4 · gradient wrt. weight ∂C∂wjk(l) = ak(l−1) δj(l)

In words: weight gradient = (activation flowing in from the upstream neuron) times (error of the downstream neuron). A connection gets credit or blame in proportion to the signal that went through it. In the worked example, w1's gradient was x × 3 = 6.

The four equations all follow from the chain rule of multivariable calculus[4], and the algorithm follows directly from them:

    Run a forward pass; cache every z(l) and a(l).
    Compute δ(L) at the output using BP1.
    For l = L − 1 down to 1: compute δ(l) using BP2.
    For each layer, accumulate gradients using BP3 and BP4.
    Update weights: W(l) ← W(l) − η · ∂C/∂W(l) and similarly for biases.

Per layer, the forward pass does one matrix-vector product and the backward pass does two (one to send δ back through the weights, one outer product for the weight gradient). A full gradient therefore costs about three forward passes, however many weights the network has, instead of one forward pass per weight. That ratio is why backprop scales to billions of parameters.
Check Your Understanding Q1 of 2

Why is backpropagation more efficient than computing each weight’s partial derivative one at a time?
It approximates the gradients, trading accuracy for speed.
It reuses cached forward-pass activations and propagates errors layer by layer in one sweep.
Modern compilers automatically vectorize it.
0/2 correct
