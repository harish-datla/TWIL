# A program is a graph of local derivatives

Training needs ∂loss/∂w, the derivative of the loss with respect to each weight w. The model computing that loss is just a program, and every program, however large, is built from small operations whose derivatives are easy: an add passes 1 to each input, a multiply's derivative toward one input is the other input, and sin's derivative is cos. Autodiff needs only those local derivatives plus a record of which operation fed which.

Draw that record with one node per operation and an arrow from each value to every operation that uses it, and you have a computational graph. The chain rule then gives a simple recipe for how the output depends on any input: follow every path from the input to the output, multiply the local derivatives along each path, and add up the paths.[1]

Worked example

f(x, y) = (x + y)·x at x = 2, y = 3. Call the sum s = x + y = 5, so f = s·x = 10.

x reaches f by two paths. Through s: ∂s/∂x = 1, then ∂f/∂s = x = 2, giving 1 × 2 = 2. Directly: ∂f/∂x with s held fixed = s = 5. Sum: ∂f/∂x = 2 + 5 = 7.

y has one path, through s: 1 × 2 = 2. Check: f = x² + xy, so ∂f/∂x = 2x + y = 7 and ∂f/∂y = x = 2.

The formula below is that recipe for any graph. Autodiff engines compute exactly this sum; the difference between them is how they organize the work.
∂f/∂x = Σpaths Πedges ∂child/∂parent

In words: the derivative of the output with respect to an input is the sum, over every path between them, of the product of the local derivatives along that path.

Listing the paths one at a time doesn't scale: their number can grow exponentially with the depth of the graph. Forward mode and reverse mode, the two strategies the next section compares, compute the same sum by merging paths wherever they meet, so each touches every edge exactly once.[1]

Reverse mode, the strategy training uses, works backwards from the output. The derivative of the final output with respect to a node is called that node's adjoint, written with a bar (ȳ). The output's adjoint is 1. Each node, once its own adjoint is complete, multiplies it by its local derivatives and adds the results into its inputs' adjoints. The engine visits nodes in reverse order of creation. That works because a node is always created after the values it uses. Walking backwards therefore reaches each node only after every node that uses it, so its adjoint is complete by the time it's passed on. In graph terms, creation order is a topological order, and the backward pass walks it in reverse.

The widget below records f(a, b) = (a·b + sin a)·b as six nodes and lets you step through that backward pass.

Worked example

At a = 1.5, b = −2 the forward pass gives a·b = −3, sin 1.5 ≈ 0.9975, their sum ≈ −2.0025, and f = −2.0025 × −2 ≈ 4.005.

Backward, starting from f̄ = 1. The final multiply sends the sum 1 × b = −2 and sends b 1 × (−2.0025). The add passes −2 unchanged to both a·b and sin a. The sin sends a −2 × cos 1.5 ≈ −2 × 0.0707 ≈ −0.1415. The product a·b sends a −2 × b = 4 and sends b −2 × a = −3.

Totals: ā = 4 − 0.1415 ≈ 3.8585 and b̄ = −2.0025 − 3 = −5.0025.
Step a backward pass through a six-node graph
a
1.5
b
-2
Step
Play
Reset
f = (a·b + sin a)·b = 4.0050 at a = 1.5, b = -2. Backward step 0 of 6: press Step to start at the output, whose adjoint is 1.

Press Step to run the backward pass one node at a time, starting at the output with adjoint 1; drag a and b to try other inputs. Each node shows its operation, its forward value, and ȳ, its adjoint so far, and the highlighted node is the one passing its adjoint to its inputs. Both a and b feed two nodes each, so their adjoints collect two contributions and are complete only after both of those nodes have been processed. After all six steps the readout compares ∂f/∂a = 3.8585 and ∂f/∂b = −5.0025 with centered finite differences, which agree to the printed four decimals.

Three local-derivative patterns cover most of a network. An add node passes its adjoint unchanged to every input. A max node, such as a ReLU, sends it to whichever input won in the forward pass and sends zero to the others. A multiply node hands each input the adjoint scaled by the other input's value. That last rule is why a weight multiplied by a large input gets a large gradient, and why rescaling inputs changes the effective learning rate, as the linear models page shows.[7]
