# The training loop, end to end

Every piece is now in place: the forward pass makes a prediction, the loss scores it, backprop finds each weight's gradient, and the optimizer moves the weights. Training runs those steps over and over on batches of examples. In pseudocode:
training_loop · pseudocode

net = makeNetwork([2, 8, 8, 2])              // shape: 2 in, 2 hidden layers of 8, 2 out
data = loadTrainingData()

for epoch in 1..N:
  shuffle(data)
  for batch in batches(data, size=32):
    batchGradients = zeros_like(net)
    for (x, y) in batch:
      // forward pass (caching every layer's activations), loss, backward pass
      exampleGradients = backprop(net, x, y)
      accumulate(batchGradients, exampleGradients)
    applyUpdate(net, batchGradients / len(batch), lr=η)
  evaluate(net, validation_set)            // loss on held-out data, see §14

Worked example

1,000 training examples, batch size 32: one pass over the data takes 32 batches (31 full ones and a last one of 8), so 32 weight updates.

50 passes: 50 × 32 = 1,600 updates, and the network sees each example 50 times.

Each pass of the outer loop over the whole dataset is one epoch. The same steps (forward pass, loss, gradient via backprop, parameter update, repeat) train everything from LeCun's 1989 USPS digit classifier[14] to today's large language models; what changes are the layer types, the optimizer, and the scale. Karpathy's Zero-to-Hero series[15] follows this loop from a single neuron up to a small GPT.
