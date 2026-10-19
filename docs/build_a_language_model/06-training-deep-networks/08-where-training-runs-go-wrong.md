# Where training runs go wrong
A loss that falls and then jumps to NaN is usually a numerics problem, not a bug in the model

Check these in order. First the learning rate against the warmup: is the peak reached too early? Then gradient clipping: is it on, and what gradient norms is it reporting? Then the precision format: float16 overflows above 65,504, and a dynamic loss scaler that keeps halving its scale means gradients keep overflowing. Only after that, suspect the architecture.

Weight decay on the wrong parameters. AdamW's weight decay acts directly on the parameters, so applying it to biases and normalization gains pulls them toward zero, which is rarely wanted. nanoGPT puts every parameter with fewer than two dimensions, meaning the biases and LayerNorm gains, in a separate group with decay turned off.[9]

Normalization can hide a bad initialization. A model that trains only because LayerNorm rescues every layer still has gradients whose size depends on the values going into each normalization. It may train slowly or need an unusually small learning rate. Check the activation statistics at initialization both before and after each normalization, as the signal widget in section 1 does.

A scaling-law fit describes one setup. It summarizes one family of runs, on one dataset, with one training recipe. Its exponents transfer better than its constants. Its optimum also shifts if serving cost is part of the budget, since a smaller model trained on more tokens is cheaper to run. Use the shape of the curve to reason, and measure the actual loss level with runs of your own.
