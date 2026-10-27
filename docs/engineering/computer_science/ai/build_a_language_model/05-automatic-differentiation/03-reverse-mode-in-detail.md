# Reverse mode in detail: adjoints accumulate

Models reuse values all the time. A recurrent network applies the same weights at every time step, and a single activation can feed several later layers. A value used in two places has two paths to the output, so its adjoint is the sum of both contributions. A reverse-mode engine therefore adds into a node's adjoint (+=) instead of assigning it (=).[7]

Worked example

f = (x + y)·x at x = 2, y = 3 again. The backward pass processes f first, which gives x̄ = 5, then the sum s, which sends x another 2.

With +=, x̄ = 5 + 2 = 7, the right answer. With =, the second contribution replaces the first and x̄ = 2. Nothing crashes and nothing is NaN; the number is just wrong.

PyTorch carries the same accumulation across calls. Each backward pass adds into the .grad of the leaf tensors, the parameters, instead of replacing it, which is why training loops zero the gradients before each backward pass.[3] Writing = where the engine needs += is a silent bug: every number still comes out finite, and the gradient is wrong for every reused value.
Reuse a value, then break the accumulation
x
1.2
w
0.8
Adjoint update
Reset
y = relu(x·w)·w + x·w at x = 1.2, w = 0.8: x feeds 2 nodes, w feeds 3. Accumulating (+=) adjoints gives ∂y/∂x = 1.4400, ∂y/∂w = 3.1200; finite differences: 1.4400, 3.1200. Largest error 9.2e-11.

Switch the adjoint update between += and = and compare the purple autograd bars with the green finite-difference bars. The function y = relu(x·w)·w + x·w uses x twice and w three times. With accumulation the tape matches finite differences to about 10⁻¹⁰. With overwriting, the last node processed wins and the earlier contributions are lost: at the defaults ∂y/∂w comes out as 0.96 instead of 3.12, and ∂y/∂x as 0.64 instead of 1.44. Drag x and w and the size of the error changes, because it depends on which contribution survives.

Frameworks such as PyTorch build the graph as the program runs, so each training step records a fresh one. A Python if statement or loop that takes a different path from one batch to the next is differentiated exactly as it ran.[3] A scalar engine of this kind, with a graph built on the fly and a backward pass in reverse order, fits in about a hundred lines; micrograd is that size.[6] Production engines work on tensors instead: one node is a whole matrix multiply with a hand-written derivative rule, rather than one node per scalar multiply.
Check Your Understanding Q1 of 2

A function has 10,000 inputs and 3 outputs. Which is cheaper for the full Jacobian?
Reverse mode: 3 passes.
Forward mode: 3 passes.
They cost the same, since each pass is about three function evaluations.
0/2 correct
