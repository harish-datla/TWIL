# Training a shared filter: add up every use

Training needs the derivative of the loss with respect to each kernel weight, so the optimizer knows which way to nudge it. The chain rule works as it does for a dense layer, which the Neural Networks page derives: a weight's derivative is how much the loss changes with each output the weight affects, times how much that output changes with the weight. What's new is sharing. A kernel weight is used at every output position, so its derivative is a sum over all of them. Code that overwrites the derivative inside the position loop, instead of adding to it, keeps only the last position and trains on the wrong number.

Worked example

The widget's input is the 3×3 grid (1, 2, 0 / 0, 1, 3 / 2, 0, 1) and the 2×2 kernel is (0.2, −0.1 / 0.3, 0.1). The four outputs are 0.1, 1.0, 0.5 and 0.

Every target is 1 and the loss is half the sum of squared errors, so the loss's derivative with respect to each output is output − 1: −0.9, 0, −0.5 and −1.

The top-left kernel weight multiplies the top-left pixel of each patch: 1, 2, 0 and 1. Its derivative is (−0.9)(1) + (0)(2) + (−0.5)(0) + (−1)(1) = −1.9, the number the code in section 07 asserts.

In general, for one input channel, stride 1 and no padding, the derivative for each kernel weight is the same sum.
∂L/∂weight[u, v] = Σrow, column
∂L/∂output[row, column] × input[row + u, column + v]

In words: for each output position, take how much the loss wants that output to change and multiply it by the pixel this weight touched there. Add those up over every position. With more input channels, each kernel gets its own sum; with a minibatch, a group of examples processed together, you also add over the examples.

To test backward code, compute the same derivative a second, independent way. A central finite difference nudges one weight up by a small step ε, runs the forward pass, nudges it down by ε, runs it again, and divides the change in loss by 2ε. No backward code is involved, so agreement between the two is strong evidence the backward pass is right.[8]
Check the shared-weight derivative
First kernel weight
0.2
Difference step
0.001
Reset
L = ½ Σ(output−1)² = 1.030000. Shared-weight analytic/numerical max difference 1.24e-13. Four output positions contribute to each derivative.

You're looking at the 2×2 forward output and, for each kernel weight, the derivative from the backward formula (purple) next to the finite-difference estimate (green). Drag the first weight or the step size and the bars keep matching; the readout shows the largest gap, about 10⁻¹³ at the defaults. This loss is quadratic in the weights, so the central difference would be exact in perfect arithmetic, and the gap is floating-point rounding.

Real networks make the check harder. If the nudge carries some value across a ReLU's corner at zero, the finite difference averages two different slopes and won't match the backward pass, even when the code is correct. A step that's too small gets lost in rounding. So check in double precision on a small case, keep values away from activation corners, and try several step sizes instead of trusting one tolerance.[9]
