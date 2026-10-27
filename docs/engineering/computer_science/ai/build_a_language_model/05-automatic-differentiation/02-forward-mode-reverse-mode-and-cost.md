# Forward mode, reverse mode, and what each costs

The same path sum can be organized two ways, and the choice decides whether the gradient of a loss with a million weights costs one pass over the graph or a million. Forward mode picks one input and carries, alongside every value, that value's derivative with respect to the chosen input. That running derivative is the tangent, written with a dot (ẋ). It's computed in the same order as the values, so one pass gives the derivative of every node with respect to one input.

Reverse mode carries adjoints backwards from one output, so one pass gives the derivative of that output with respect to every node.[1]

Worked example

f = (x + y)·x at x = 2, y = 3, with s = x + y = 5, as before.

Forward mode, seeded with ẋ = 1, ẏ = 0: ṡ = ẋ + ẏ = 1, then ḟ = ṡ·x + s·ẋ = 1×2 + 5×1 = 7. That's ∂f/∂x only. Getting ∂f/∂y = 2 takes a second pass seeded with ẋ = 0, ẏ = 1.

Reverse mode, seeded with f̄ = 1: s̄ = x = 2 and x̄ gets s = 5 directly; then s passes 2 to both inputs, so x̄ = 5 + 2 = 7 and ȳ = 2. One pass, both derivatives.

The two update rules below are what each mode runs at every node.
ċ = Σp (∂c/∂p) ṗ    p̄ = Σc c̄ (∂c/∂p)

In words: in forward mode a node's tangent is the sum, over its inputs, of the local derivative times that input's tangent. In reverse mode a node's adjoint is the sum, over the nodes that use it, of their adjoint times the local derivative. Same products, read from opposite ends.

A function with n inputs and m outputs has n × m first derivatives, and the grid holding them all is its Jacobian. One forward pass fills one column of that grid (every output's derivative with respect to one input), which is why it's called a Jacobian-vector product. One reverse pass fills one row (one output's derivative with respect to every input), a vector-Jacobian product. So the full Jacobian costs n forward passes or m reverse passes. By JAX's count each pass costs about three times one evaluation of the function: one unit to evaluate it, one to compute the local derivatives, one to apply them.[2]

A training loss maps millions of parameters to one number: n is huge and m is 1. Reverse mode gets the whole gradient in a single pass, while forward mode would need one pass per parameter.[2] That's why reverse mode, under the name backpropagation, is the default for training. For a function with few inputs and many outputs, forward mode wins by the same margin.[1]
Count the work for a full Jacobian both ways
Inputs n
8
Outputs m
2
Reset
Graph: 8 inputs → 16 hidden units → 2 outputs, 496 edges. Full Jacobian by forward mode: 8 passes, 3968 multiply-adds. By reverse mode: 2 passes, 944 multiply-adds. Ratio forward/reverse 4.20. Entry ∂y₀/∂x₀ agrees both ways: -0.431193 vs -0.431193.

Slide the number of inputs n and outputs m and compare the bars: the left panel counts the multiply-adds each mode spends on the full Jacobian, and the right counts the passes. The graph is a seeded two-layer scalar network with 16 hidden units, built on the same tape as the other widgets. The widget really runs n forward passes and m reverse passes and counts their work; nothing is estimated from a formula. At the default 8 inputs and 2 outputs, forward mode does 4.2 times the work of reverse mode. At 32 inputs and 1 output (the shape of a loss function) it does 32 times the work. At 1 input and 8 outputs the ratio flips to 0.2. The readout also computes one Jacobian entry both ways, and the two match.
