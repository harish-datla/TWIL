# Your turn: the code playground

The playground below runs the same compact JavaScript library as the trainer, exposed as the MPGNet object, so you can build your own network, train it, and predict from the editor. The Python, C++, and Rust ports from §12 do the same thing; copy any of them into your own editor to run there. Hit RUN (or Ctrl+Enter / Cmd+Enter) to execute; output prints on the right.
▸ playground.js · live JavaScript, in your browser
Reset
▶ Run
// Build a network: 2 inputs → 6 hidden → 6 hidden → 2 outputs
const net = MPGNet.makeNetwork([2, 6, 6, 2], { activation: 'relu' });

// Tiny XOR dataset: [x1, x2, label], label 1 when the signs differ
const data = [
  [-0.8, -0.8, 0], [ 0.8,  0.8, 0],
  [-0.8,  0.8, 1], [ 0.8, -0.8, 1],
];

// Train for 800 epochs
const learningRate = 0.05;
for (let epoch = 0; epoch < 800; epoch++) {
  let totalLoss = 0;
  for (const [x1, x2, label] of data) {
    const gradients = MPGNet.backprop(net, [x1, x2], label);   // { dW, dB, loss, probs }
    MPGNet.applyGrads(net, gradients, learningRate);
    totalLoss += gradients.loss;
  }
  if (epoch % 100 === 0) {
    console.log(`epoch ${epoch.toString().padStart(3)} · loss ${(totalLoss / data.length).toFixed(4)}`);
  }
}

console.ok('training complete');

// Test on the training points
for (const [x1, x2, label] of data) {
  const probs = MPGNet.predict(net, [x1, x2]);
  const predicted = probs[0] > probs[1] ? 0 : 1;
  const tag = predicted === label ? '✓' : '✗';
  console.log(`${tag}  (${x1}, ${x2}) → class ${predicted}  · probs [${probs[0].toFixed(2)}, ${probs[1].toFixed(2)}]  (true: ${label})`);
}

// Test a held-out point
const heldOut = MPGNet.predict(net, [0.5, -0.5]);
console.log(`\nheld-out (0.5, -0.5) → class ${heldOut[0] > heldOut[1] ? 0 : 1}`);
press ▶ RUN (or Ctrl+Enter) to execute. Use MPGNet for the built-in NN library.

Edit anything: the data, the architecture, the learning rate, a held-out test set. The library follows the §12 JavaScript with shorter names: makeNetwork(layerSizes, { activation }) returns { W, B } (weights and biases), backprop(net, input, classIndex) returns { dW, dB, loss, probs }, and forward, predict, applyGrads and softmaxVec round it out. It also lives at window.MPGNet for use from the browser console.
