# Smoothing it out: the sigmoid neuron

Training adjusts weights in small steps and needs to know whether each small step helped. With a hard threshold it can't tell. Push z from −0.001 to +0.001 and the output jumps from 0 to 1; anywhere else, a small weight change moves the output by exactly nothing. There's no "almost firing." In calculus terms, the step's slope is zero everywhere except at the jump, so a learning rule that follows the slope gets no signal.

The fix is to replace the jump with a smooth S-shaped curve, so a small change in z always makes a small change in the output. Rumelhart, Hinton and Williams used the logistic sigmoid[2]. It squashes any number into the range 0 to 1: very negative inputs land near 0, very positive ones near 1, and zero lands exactly in the middle.

Worked example

z = 0: 1 / (1 + e0) = 1 / 2 = 0.5.

z = 2: e−2 ≈ 0.135, so 1 / 1.135 ≈ 0.881.

z = −2: e2 ≈ 7.39, so 1 / 8.39 ≈ 0.119. The two answers add to 1, because the curve is symmetric around z = 0.

The sigmoid computes that squashed value, which a network reads as how strongly the neuron fires:
eq. 3 · sigmoid σ(z) = 11 + e−z

In words: raise e (about 2.718) to the power −z, add 1, and take one over the result. Large positive z gives nearly 1, very negative z gives nearly 0, and z = 0 gives exactly 0.5. Around z = 0 the curve is gently sloped, so nudging z a little nudges the output a little.

Training also needs the curve's slope at each point, because backpropagation (§9) multiplies by it to work out how much each weight is to blame. At z = 0 the slope is 0.5 × (1 − 0.5) = 0.25. At z = 2 it's 0.881 × 0.119 ≈ 0.105. Both come from one short formula for the slope, written σ′ ("sigma prime"):
eq. 4 · sigmoid derivative σ′(z) = σ(z) · (1 − σ(z))

In words: the slope of the sigmoid at z is the sigmoid's value there times one minus that value. Code that has already computed σ(z) gets the slope with one multiply.

The slope is largest, 0.25, at z = 0, the curve's steepest point (its inflection point), and it decays toward zero in both directions. Turn on "show derivative" in the widget below and slide the input to see it.

Live · Activation Functions
sigmoid
tanh
ReLU
leaky
GELU
input z 0.00
show derivative f′(z)
f(z)
0.5000
f′(z)
0.2500

f′(z) is the slope of the curve: how much the output changes per unit of input. Training needs it to know which way to nudge the weights.
The solid purple curve is the selected activation function and the dashed blue curve is its slope; slide z to move the marker along both, and use the buttons to switch functions. At the default, sigmoid at z = 0, the readout shows f(z) = 0.5 and f′(z) = 0.25. Sigmoid and tanh are S-shaped, ReLU and leaky ReLU are piecewise-linear, and GELU is a smoothed ReLU. All five have a slope defined everywhere (ReLU and leaky ReLU have a kink at 0, where code picks one side's slope), but ReLU's is zero for every negative input. §10 explains why those differences matter for deep networks.
