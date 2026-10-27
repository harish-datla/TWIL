# Learning by descent

The loss turns the network's weights into one number, and training means finding weights that make that number small. A network with millions of weights can't try combinations one by one. The standard strategy is gradient descent: repeatedly move every weight a little in whichever direction lowers the loss.

Picture standing on a hillside in dense fog. You can't see the bottom of the valley, but you can feel the slope under your feet. The gradient is that slope written as a vector: it points uphill, so you step the opposite way, feel the slope again, and step again. Repeat until the ground is flat, which is a low point, though not necessarily the lowest one.

Worked example

One weight, with loss C(w) = (w − 3)². The best weight is 3. Start at w = 1, where the loss is 4.

The slope there is 2(w − 3) = −4: negative, so increasing w lowers the loss. Step size η = 0.1: w becomes 1 − 0.1 × (−4) = 1.4, and the loss drops to (1.4 − 3)² = 2.56. The next step reaches 1.72 and a loss of about 1.64.

With η = 1.1 instead, the first step goes to 1 + 1.1 × 4 = 5.4. That's past 3, and the loss rises to 5.76: the step overshot the bottom.

With many weights, do the same thing for each one. For a weight w, compute ∂C / ∂w, how much the loss changes per unit change in that weight with every other weight held still. (If the curly-d notation is new to you, expand the box below.) The update rule then moves each weight a small step against its slope:

eq. 9 · gradient descent update w ← w − η · ∂C∂w

In words: the arrow means "assign." Replace each weight with itself minus η times its slope. Do this for every weight, every step, and the loss goes down, provided η is small enough.

The constant η ("eta") is the learning rate, how big a step to take. Too small and training crawls; too big and each step overshoots the valley floor, as the η = 1.1 step in the example did, so the weights oscillate or diverge. It's usually the first setting to tune.

Click anywhere on the loss surface below to drop a starting point, then hit DESCEND; the yellow path traces each step. On the elongated valley the loss is ½(w² + 12b²), so its slope across the valley grows 12 times faster than along it. The path bounces between the walls once η passes 1/12 ≈ 0.08 and flies off once it passes 2/12 ≈ 0.17 (the default 0.18 diverges there; lower it and descend again). The other surfaces show a saddle point and two separate minima:
Live · Gradient Descent
loss surface
learning rate η 0.18
Descend ▶
Reset ⟲
steps
0
loss
3.2800

Click on the surface to place a starting point. Bright regions = low loss, dark = high. Green dots mark the minima. On the saddle surface, start close to the b = 0 line: the path slows to a crawl near the saddle point before tipping into one of the two wells. On the two-minima surface, which well you end in depends only on where you start.
You're looking down on a loss surface over two weights, w and b: bright is low loss, dark is high. Click to place a start, press Descend, and the yellow path traces each step; the readout counts steps and shows the current loss. A real network's surface has one dimension per weight, millions of them instead of two, and gradient descent never sees the whole surface, only the slope at its current point.
Stochastic gradient descent

Computing the gradient over the entire training set is expensive. The variant that makes neural nets practical is stochastic gradient descent (SGD): estimate the gradient from a small random batch of examples (a "mini-batch"), update the weights, then grab another batch. The estimate is noisy, which is acceptable, and the noise can help the optimizer move off saddle points and out of sharp local minima.
The same surface, in three dimensions

The contour map above is the usual way to draw it on paper. With the loss as height, the same kind of surface becomes a 3D terrain. Drag the surface below to rotate it, then hit Descend to see the path slide down the side of the hill:
Live · 3D Loss Surface drag to rotate
surface
learning rate η 0.150
Descend ▶
Reset ⟲

Drag the surface to rotate it. The brighter the color, the lower the loss. The yellow trace shows the path of gradient descent.
The height of the surface is the loss over two weights; drag to rotate it, pick a surface, and press Descend to send the yellow path downhill. On the double well, descent starts at w = 0.1, just past the ridge between the two minima, so it rolls into the nearer well and never visits the other one: which minimum gradient descent finds depends on where it starts.
Check Your Understanding Q1 of 2

What happens to the descent trajectory if you set the learning rate η too large?
It converges faster to a better solution.
Each step lands past the valley floor, and the trajectory oscillates or diverges.
Nothing changes; the gradient direction is what matters.
0/2 correct
