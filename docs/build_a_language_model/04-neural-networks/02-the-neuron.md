# The neuron: a tiny decision maker

Start with a smaller decision. A spam filter looks at a few measurements of an email, say how many links it contains and whether the sender is in your contacts, and has to answer yes or no. An artificial neuron makes exactly this kind of decision. It multiplies each measurement by a number that says how much that measurement matters, adds up the results, and answers yes if the total clears a bar.

Those per-input numbers are the neuron's weights. A positive weight pushes toward yes, a negative one pushes toward no, and a weight near zero means the input barely counts. The neuron also has one extra number, its bias, which is added to the total and sets how high the bar is.

Worked example

Inputs: x1 = 3 links, x2 = 0 (sender not in your contacts). Weights: w1 = 0.5, w2 = −2. Bias: b = −1.

Total: 0.5×3 + (−2)×0 + (−1) = 0.5. That's above zero, so the neuron says spam.

The same email from a contact (x2 = 1): 1.5 − 2 − 1 = −1.5, so it says not spam. The large negative weight on "known sender" outvotes the links.

That total is the neuron's pre-activation, written z. With inputs x1, x2, …, xn, weights w1, w2, …, wn and bias b, it's one multiply per input, all added up, plus the bias:
eq. 1 · pre-activation z = w1x1 + w2x2 + … + wnxn + b

In words: multiply each input by its weight, add the products, then add the bias. The result z is a single number. Hover any letter in the equation to see what it stands for.

Multiplying matching entries of two arrays and adding them up is a dot product, so in vector form z = w · x + b: a few lines of code in any language.

The neuron then turns z into its output with an activation function. Rosenblatt's perceptron, the first artificial neuron with a learning rule[1], used the simplest one there is: a hard threshold that turns the score into a yes or a no.
eq. 2 · perceptron output output = 1  if  z ≥ 00  if  z < 0

In words: if z is zero or positive, the neuron fires and outputs 1. If z is negative, it stays silent and outputs 0. The whole rule is one if-statement.

Two inputs make the rule easy to draw. Plot each input pair as a point on a plane. The points where w1x1 + w2x2 + b = 0 form a straight line. Points on one side give z > 0 and the neuron fires; points on the other side give z < 0 and it stays silent. Changing a weight tilts the line, and changing the bias slides it without changing its angle.

The widget starts with w1 = w2 = 1 and b = −0.3, and its dot sits at (0.4, 0.6), so z = 0.4 + 0.6 − 0.3 = 0.7 and the neuron fires. Drag the dot across the line and the output flips.
Live · Perceptron
w1 · weight on x1 1.00
w2 · weight on x2 1.00
b · bias -0.30
z = w·x + b
0.700
output
fires (1)
Drag the dot, or move a slider. The shaded background shows where the neuron would fire (purple) or stay silent (blue), and the straight purple line between them is the decision boundary, where z = 0. The readout gives z for the dot and whether the neuron fires.

So a single perceptron draws one straight line and can only answer by which side of it a point is on. It can separate two groups of points only if they're linearly separable, meaning one straight line divides them. No choice of weights makes a single perceptron compute XOR, which outputs 1 when exactly one of its two inputs is 1[3]. That limit is why networks stack neurons in layers (§4). First, though, the hard threshold has a more immediate problem.
Check Your Understanding Q1 of 2

You set w1 = 2, w2 = 0, and b = −1. The neuron looks only at the first input. For which input does it fire?
x1 = 0.3, x2 = 5
x1 = 0.8, x2 = −10
x1 = 0.1, x2 = 100
0/2 correct
