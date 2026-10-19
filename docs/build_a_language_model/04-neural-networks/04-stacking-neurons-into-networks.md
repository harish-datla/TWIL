# Stacking neurons into networks

One straight line is enough when the two classes sit on opposite sides of a line. Plenty of data doesn't: points of one class can form a ring around the other, or the two classes can wind around each other as two interleaved spirals, the dataset the trainer in §13 learns. No single line separates those.

Put two neurons side by side and you get two lines. Add a second layer of neurons that takes the first layer's outputs as its inputs, and it can combine those cuts into a curved region. With enough neurons, that region can follow a circle or a spiral. A feedforward network is built this way: layers of neurons, where every neuron in one layer takes inputs from every neuron in the layer before. The first layer is the input layer (it just holds the input numbers), the last is the output layer, and the ones in between are hidden layers, named because their outputs aren't part of the network's answer.
input hidden output

The diagram above is a 2-3-2 network: two inputs, three hidden neurons, two outputs. Every line is one weight, and every circle except the inputs holds one bias. The whole thing is a function that turns two input numbers into two output numbers.

Writing every neuron's sum out by hand gets long, so code and math group a layer's numbers. Put the layer's weights in a grid, a matrix W, with one row per neuron and one column per input. Put its biases in a list b. Each neuron's pre-activation is then the dot product of its row with the input, plus its bias.

Worked example

The hidden layer of the forward-propagation widget in §5. Weight rows: (0.9, −0.6), (−0.4, 1.1), (0.6, 0.5). Biases: (0.1, −0.2, 0). Input: x = (0.6, 0.4).

Neuron 1: 0.9×0.6 + (−0.6)×0.4 + 0.1 = 0.54 − 0.24 + 0.1 = 0.40.
Neuron 2: −0.24 + 0.44 − 0.2 = 0.00.
Neuron 3: 0.36 + 0.20 + 0 = 0.56.

Sigmoid of each: σ(0.40) ≈ 0.599, σ(0) = 0.5, σ(0.56) ≈ 0.636. Those three numbers are the layer's output, and they become the next layer's input.

For layer number l, write its weight matrix W(l), its biases b(l), and its outputs (its activations) a(l). These two lines compute one layer's outputs from the previous layer's, which is all a network does to make a prediction:
eq. 5 · layer forward pass z(l) = W(l) a(l−1) + b(l) a(l) = σ(z(l))

In words: multiply the previous layer's outputs by this layer's weight matrix (each row gives one neuron's weighted sum), add the biases to get the pre-activations z, then apply the activation σ to each entry separately. The superscript (l) only labels which layer. A whole network is a few of these steps in a row.

Can one hidden layer learn anything?

Check Your Understanding Q1 of 2

Why does stacking neurons into multiple layers give the network more expressive power than a single neuron, even though each individual neuron is still just a weighted sum plus an activation?
Stacking adds parameters, and more parameters always mean more accuracy.
Each hidden layer reshapes the input space, so later layers see a transformed problem where curved boundaries become straight again.
Multiple layers vote on the answer and majority wins.
0/2 correct
