# The engine trains a network

The scalar tape from the first section is enough to train a real network, if a tiny one. XOR is the classic test: output 1 when exactly one of two inputs is 1, and 0 otherwise. No straight line separates those cases, so a linear model can't learn it, but a network with a hidden layer can. The widget builds a network with 2 inputs, 2 hidden ReLU units and 1 output, 9 parameters in all, on the tape, one node per scalar operation. Each step it computes the squared-error loss over the four XOR cases, runs the same reverse pass as in the first section, and moves the nine parameters downhill. The readout also checks the gradient against finite differences at the current weights, every step.
Train XOR with the tape and check every gradient
Learning rate η
0.3
Step
Play
Reset
0 gradient steps at η = 0.3, 9 parameters, loss = mean squared error over the four XOR cases: 0.52468. Outputs for (0,0), (0,1), (1,0), (1,1): 0.160, -0.410, 0.730, -0.110. Gradient check at the current weights: largest |autodiff − finite difference| = 2.5e-2. Tape size per loss evaluation: 89 nodes.

Press Play to train; each Step applies ten gradient updates from a fixed start. The left panel shows the loss, and the right shows the network's output over the unit square, with x₂ pointing up. The loss falls from 0.52 to below 10⁻³ within 60 updates at the default rate, and the output map develops the diagonal pattern XOR needs: high at (0, 1) and (1, 0), low at the other two corners. Each loss evaluation builds a tape of 89 nodes. The gradient check reads 2.5·10⁻² at Step 0: there the (1, 1) case puts the first hidden unit's input at 0.6 − 0.7 + 0.1, which is 0 in exact arithmetic and about 3·10⁻¹⁷ after rounding, right on a corner. On Steps 1 to 10 the check reads 10⁻¹² or less. Training then drives the (0, 0) and (1, 1) inputs of both hidden units toward zero, inside h of the corner, so from Step 11 the check jumps to around 10⁻⁵ and settles near 3·10⁻⁶ by Step 15. That's the corner effect from the previous section, with both derivatives correct.
Check Your Understanding Q1 of 2

A 96-layer model runs out of memory during backward. Checkpointing every 12 layers would change the peak activation count to roughly what?
About 20, with roughly 90% extra forward compute.
About 12, with no extra compute.
About 48, because checkpointing halves memory.
0/2 correct
