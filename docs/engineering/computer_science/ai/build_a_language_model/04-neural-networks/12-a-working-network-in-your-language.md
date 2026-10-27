# A working network, in your language

Below is the complete network (initialization, forward pass, backprop, SGD update) in four languages: JavaScript, Python, C++, and Rust. Pick whichever you're most comfortable with. The algorithm is identical in all four; only the syntax changes. Every variable has a descriptive name, and every non-obvious line gets a comment.
Following along in an editor?

Copy any version into a file with the matching extension. The JavaScript version works in Node.js or any browser. The Python version is pure stdlib (no NumPy). The C++ version compiles with -std=c++11 or later. The Rust version uses only std with a tiny hand-rolled pseudo-random number generator, or PRNG (in real code, use the rand crate). Each is roughly 150 to 200 lines. They are libraries: add a main or a training loop like the one in §11 to run them.
network · pick a language ↓
JavaScript
Python
C++
Rust
Copy

// ──────────────────────────────────────────────────────────────
// network.js · feed-forward neural net trained with backprop + SGD
// ──────────────────────────────────────────────────────────────

// Activation functions, paired with their derivatives.
// Note: each derivative is expressed in terms of the *post-activation*
// value `a`, not the pre-activation `z`. For our three choices this
// makes the derivative cheap to compute from values we already have.
const ACTIVATIONS = {
  relu: {
    fn:                       z => z > 0 ? z : 0,
    derivativeFromActivation: a => a > 0 ? 1 : 0,
  },
  tanh: {
    fn:                       z => Math.tanh(z),
    derivativeFromActivation: a => 1 - a * a,
  },
  sigmoid: {
    fn:                       z => 1 / (1 + Math.exp(-z)),
    derivativeFromActivation: a => a * (1 - a),
  },
};

// Build a fresh network. `layerSizes` is an array of integers,
// e.g. [2, 8, 8, 2] = 2 inputs, two hidden layers of 8, 2 outputs.
function makeNetwork(layerSizes, activationName = 'relu') {
  const weights = [];   // weights[layer][neuron][input]
  const biases  = [];   // biases[layer][neuron]

  for (let layer = 1; layer < layerSizes.length; layer++) {
    const fanIn  = layerSizes[layer - 1];   // inputs coming in
    const fanOut = layerSizes[layer];       // neurons in this layer

    // Target weight variance: 2/fanIn for ReLU (He init), 1/fanIn otherwise
    // (Xavier-style). Both keep activations from blowing up or vanishing
    // with depth. A uniform draw on [-limit, limit] has variance limit²/3,
    // so limit = sqrt(3 × targetVariance).
    const targetVariance = activationName === 'relu' ? 2 / fanIn : 1 / fanIn;
    const limit = Math.sqrt(3 * targetVariance);

    const layerWeights = [];
    for (let neuron = 0; neuron < fanOut; neuron++) {
      const weightRow = [];
      for (let input = 0; input < fanIn; input++) {
        weightRow.push((Math.random() * 2 - 1) * limit);
      }
      layerWeights.push(weightRow);
    }
    weights.push(layerWeights);
    biases.push(new Array(fanOut).fill(0));   // start biases at zero
  }

  return { layerSizes, weights, biases, activationName };
}

// Forward pass: turn an input vector into an output prediction.
// Returns both the activations (per layer) and the pre-activations,
// because we'll need both for backprop.
function forward(network, inputVector) {
  const activationsPerLayer    = [inputVector.slice()];
  const preActivationsPerLayer = [];
  const activation = ACTIVATIONS[network.activationName];

  for (let layer = 0; layer < network.weights.length; layer++) {
    const layerWeights        = network.weights[layer];
    const layerBiases         = network.biases[layer];
    const previousActivations = activationsPerLayer[activationsPerLayer.length - 1];

    const preActivations    = [];
    const currentActivations = [];
    const isOutputLayer = (layer === network.weights.length - 1);

    for (let neuron = 0; neuron < layerWeights.length; neuron++) {
      // Weighted sum + bias = pre-activation `z`.
      let z = layerBiases[neuron];
      const weightRow = layerWeights[neuron];
      for (let input = 0; input < weightRow.length; input++) {
        z += weightRow[input] * previousActivations[input];
      }
      preActivations.push(z);

      // Output layer stays as raw logits, we apply softmax separately
      // when we need probabilities. Hidden layers use the activation fn.
      currentActivations.push(isOutputLayer ? z : activation.fn(z));
    }

    preActivationsPerLayer.push(preActivations);
    activationsPerLayer.push(currentActivations);
  }

  return { activationsPerLayer, preActivationsPerLayer };
}

// Softmax: turn the raw output logits into a probability distribution.
// Subtracting the max first is a numerical-stability trick, same answer,
// no overflow.
function softmax(logits) {
  const maxLogit = Math.max(...logits);
  const exponentials = logits.map(z => Math.exp(z - maxLogit));
  const total = exponentials.reduce((sum, value) => sum + value, 0);
  return exponentials.map(value => value / total);
}

// Backpropagation for softmax + cross-entropy loss.
// Returns the gradient of the loss with respect to every weight and
// bias in the network, plus the loss itself.
function backprop(network, inputVector, trueClassIndex) {
  const { activationsPerLayer } = forward(network, inputVector);
  const numLayers = network.weights.length;
  const activation = ACTIVATIONS[network.activationName];

  // Allocate zero-initialized gradient arrays with the same shape
  // as the weights and biases.
  const weightGradients = network.weights.map(layer =>
    layer.map(row => row.map(() => 0))
  );
  const biasGradients = network.biases.map(layer => layer.map(() => 0));

  // BP1: at the output, the error simplifies (for softmax+CE) to
  //      (predicted probability) − (one-hot true label).
  const outputLogits     = activationsPerLayer[numLayers];
  const outputProbabilities = softmax(outputLogits);
  let errorAtCurrentLayer = outputProbabilities.slice();
  errorAtCurrentLayer[trueClassIndex] -= 1;

  // Walk backward through the layers, computing gradients as we go.
  for (let layer = numLayers - 1; layer >= 0; layer--) {
    const previousLayerActivations = activationsPerLayer[layer];

    // BP3 + BP4: gradient of the loss wrt this layer's biases and weights.
    for (let neuron = 0; neuron < network.weights[layer].length; neuron++) {
      biasGradients[layer][neuron] = errorAtCurrentLayer[neuron];
      for (let input = 0; input < previousLayerActivations.length; input++) {
        weightGradients[layer][neuron][input] =
          errorAtCurrentLayer[neuron] * previousLayerActivations[input];
      }
    }

    // BP2: propagate the error backward to the previous layer,
    // scaled by the activation function's derivative.
    if (layer > 0) {
      const errorAtPreviousLayer = new Array(network.weights[layer - 1].length).fill(0);
      for (let source = 0; source < errorAtPreviousLayer.length; source++) {
        let weightedError = 0;
        for (let target = 0; target < network.weights[layer].length; target++) {
          weightedError += network.weights[layer][target][source] * errorAtCurrentLayer[target];
        }
        errorAtPreviousLayer[source] =
          weightedError * activation.derivativeFromActivation(activationsPerLayer[layer][source]);
      }
      errorAtCurrentLayer = errorAtPreviousLayer;
    }
  }

  // Cross-entropy loss for this example.
  const loss = -Math.log(Math.max(1e-9, outputProbabilities[trueClassIndex]));
  return { weightGradients, biasGradients, loss, outputProbabilities };
}

// One SGD step: subtract a small multiple of each gradient.
function applyGradients(network, gradients, learningRate) {
  for (let layer = 0; layer < network.weights.length; layer++) {
    for (let neuron = 0; neuron < network.weights[layer].length; neuron++) {
      network.biases[layer][neuron] -= learningRate * gradients.biasGradients[layer][neuron];
      for (let input = 0; input < network.weights[layer][neuron].length; input++) {
        network.weights[layer][neuron][input] -=
          learningRate * gradients.weightGradients[layer][neuron][input];
      }
    }
  }
}

What's intentionally missing

The listings trade speed and robustness for readability. A production implementation would add:

    Vectorized math. Nested loops over nested arrays instead of contiguous matrices and a BLAS or GPU matrix multiply, which run orders of magnitude faster.
    Batched training. backprop handles one example and allocates fresh gradient arrays each call; real code processes a mini-batch as one matrix and reuses buffers.
    Reuse of the forward pass. backprop runs its own forward pass, so a caller that already ran forward pays for it twice, and the pre-activations forward returns go unused because every derivative here can be computed from the post-activation.
    A better optimizer and regularization. Plain SGD only: no momentum, Adam, weight decay, or learning-rate schedule (§8, §14).
    Numerical guards. Softmax subtracts the max logit, but the sigmoid does not guard its exp: Python's math.exp raises OverflowError for arguments above about 709, so a pre-activation below −709 crashes the Python version.
    Reproducibility and I/O. No fixed random seed in JavaScript or Python, no saving or loading of trained weights, no data pipeline.

Every line traces back to an equation from earlier sections. Large language models are trained with the same loop of forward pass, loss, backprop and update, scaled up with matrix libraries, GPUs, attention layers, better optimizers and billions of parameters.
