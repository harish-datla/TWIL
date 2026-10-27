# Kinks, conventions and gradient checks

ReLU, max(0, x), has a corner, or kink, at 0, where its slope jumps from 0 to 1. Autodiff is exact only where every primitive has a derivative, and at a corner like that one (or sqrt at 0) there isn't one. The framework still has to return something. PyTorch documents its choice, and for ReLU at exactly 0 the answer is 0.[3] The choice rarely matters for training, because landing exactly on a corner in floating point is uncommon. It does mean the "gradient" of a piecewise-linear network is a convention at its corners.
PyTorch's full rule for non-differentiable points

A finite-difference check, the slope between the loss at w + h and at w − h from the linear models page, sees corners differently. If a corner sits within h of the test point, the centered difference straddles it and averages the slopes on the two sides. It then disagrees with autodiff even though both are doing their job correctly.

Worked example

g(w) = relu(w − 1) + ½·relu(−w), the widget's function. Its slope is 0 just left of w = 1 and 1 just right of it.

At w = 1 with h = 0.001: (g(1.001) − g(0.999)) / 0.002 = (0.001 − 0) / 0.002 = 0.5, the average of the two slopes. Autodiff, using ReLU's slope of 0 at the corner, reports 0.

So test gradients at ordinary points, away from corners, and when a check fails, move the point before concluding that the derivative rule is wrong.
Compare autodiff and finite differences across two kinks
w
1
h = 10⁻ᵏ, k
3
Reset
g(w) = relu(w − 1) + ½·relu(−w) at w = 1: g = 0.0000; autodiff derivative 0.0000 (relu' taken as 0 at the kink); centered difference with h = 0.001 gives 0.5000. On the plotted grid the two disagree at 2 of 251 points, all within h of a kink at w = 0 or w = 1.

Drag w across the two corners at w = 0 and w = 1, and change h. The green curve is the autodiff derivative, from a reverse pass on the tape at each plotted point; the yellow curve is the centered finite difference. Away from the corners yellow sits on green. Within h of a corner the finite difference reports an average slope: exactly at w = 1, autodiff reports 0 by the convention above while the finite difference reports 0.5. At the default h = 10⁻³ the two disagree at 2 of the 251 plotted points, and at h = 10⁻¹ at 18. Shrinking h narrows the band of disagreement but never removes the corner itself.
