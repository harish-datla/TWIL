# Forward propagation

Making a prediction means running the layers in order. The input feeds the first hidden layer, that layer's outputs feed the next, and the last layer's outputs are the answer. This is forward propagation: apply the two layer equations from §4 one layer at a time and pass the activations forward.

For a classifier, the answer should be one probability per class. The output layer below therefore applies softmax instead of sigmoid: softmax turns its list of output numbers into probabilities that are all positive and add up to 1 (§6 shows how).

Worked example

Continue §4's example. The hidden outputs (0.599, 0.5, 0.636) go into the output layer, whose weight rows are (0.8, −0.7, 0.5) and (−0.5, 0.6, 0.9), with biases (0, 0.1).

Output 1: 0.8×0.599 − 0.7×0.5 + 0.5×0.636 ≈ 0.447.
Output 2: −0.5×0.599 + 0.6×0.5 + 0.9×0.636 + 0.1 ≈ 0.673.

Softmax turns (0.447, 0.673) into probabilities (0.44, 0.56), so the network picks class 2, but without much confidence.

The widget runs that same pass. Set the inputs, then click STEP to advance one phase at a time, or RUN to animate the whole pass:
Live · Forward Propagation 0 · inputs fed in.
0.60x10.40x2ŷ1ŷ2INPUTHIDDENOUTPUT

Forward-propagation visualization

An animated two-to-three-to-two network. Each step lights up the next phase: feeding inputs in, computing hidden pre-activations, applying the activation function, computing output logits, and finally applying softmax. The current stage is also reported in the text status line.
x1 · input 1
x2 · input 2
Step ▸
Run ▶
Reset ⟲
click STEP to advance, or RUN to animate.
Each click of Step lights up the next phase, left to right; the status line names the phase. Edge thickness shows a weight's size, purple for positive and blue for negative, and the number on each edge is the weight. Node fill shows the activation value. At the default inputs (0.6, 0.4) the pass ends at probabilities [0.44, 0.56], the worked example's answer; move the sliders to see the prediction change.

In code, forward propagation is a short loop over the layers. This sketch applies sigmoid at every layer for brevity; the full listing in §12 leaves the output layer as raw numbers for softmax.
forward.js · pseudo-implementation
Copy

function forward(network, x) {
  let a = x;                              // activations of the previous layer
  const activations = [a];                // keep every layer's activations for backprop

  for (let l = 0; l < network.W.length; l++) {
    const W = network.W[l];               // weight matrix for this layer
    const b = network.B[l];               // bias vector for this layer
    const z = matVecMul(W, a).map((zi, i) => zi + b[i]);
    a = z.map(sigmoid);                  // element-wise activation
    activations.push(a);
  }
  return activations;                      // last entry is the network's output
}

activations[activations.length - 1] is the prediction. The function keeps every layer's activations because backpropagation (§9) reuses them to compute the gradients.
