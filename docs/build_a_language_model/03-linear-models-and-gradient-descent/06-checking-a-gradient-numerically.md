# Checking a gradient numerically

You've worked out a gradient by hand, coded it, and the loss goes down. That doesn't prove the gradient is right. A dropped factor of 2, or a wrong sign on one weight among many, can still leave the loss falling for a while, so the bug hides. The fix is to estimate the gradient a second way, with no calculus, and compare the two.

Nudge one weight up by a tiny amount h and evaluate the loss. Nudge it down by h and evaluate again. The difference divided by 2h is the slope between those two nearby points, a finite difference, and it should match that weight's entry in your gradient.[9]

Worked example

f(w) = w³ at w = 2. The true slope is 3 × 2² = 12.

Centered, h = 0.1: (2.1³ − 1.9³) / 0.2 = (9.261 − 6.859) / 0.2 = 12.01, off by 0.01.

One-sided, h = 0.1: (2.1³ − 2³) / 0.1 = (9.261 − 8) / 0.1 = 12.61, off by 0.61.

The centered version, which steps both ways, is far more accurate, because its error shrinks like h² instead of like h. The formula below is the check the code at the end of this page runs for each weight.
∂L/∂w ≈ (L(w + h) − L(w − h)) / 2h

In words: the gradient entry for one weight is close to the rise in the loss between a point just below that weight and a point just above it, divided by the distance between them.

A step around h = 10⁻⁵ is a common working choice.[9] Going much smaller makes the estimate worse. A double holds roughly 16 significant digits; with a tiny h the two losses agree in most of them, and subtracting them leaves mostly rounding error. The floating point page explains why.
Sweep the step size of a finite-difference check
h = 10⁻ᵏ, k
5
Reset
h = 10⁻5: centered difference (0.044921, -0.551352, -0.274697) vs analytic (0.044921, -0.551352, -0.274697); largest difference 1.21e-11. Smallest error on this sweep at h = 10⁻5 (1.21e-11). Large h: truncation error; tiny h: rounding error from subtracting nearly equal losses.

Drag k to set the step h = 10⁻ᵏ. The left panel plots, for every h from 10⁻¹ to 10⁻¹², the largest gap between the finite-difference gradient and the hand-derived one; the bars on the right compare the two gradients component by component at the h you picked. The loss is the logistic loss from the section above at a fixed weight vector, in double precision. The curve is V-shaped on this log scale. For large h the slope estimate itself is coarse, and the error falls 100-fold for each step of k, as h² predicts, from 1.6 × 10⁻⁴ at h = 10⁻¹. Below h = 10⁻⁵ rounding takes over and the error climbs again. The best agreement, about 1.2 × 10⁻¹¹, is at h = 10⁻⁵. An error that falls as h² for large h and bottoms out far below the size of the gradient (here about 0.04 to 0.55) means the hand-derived gradient is right; an error that stays large at every h means it's wrong.
