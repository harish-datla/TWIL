# Momentum and Adam

Gradient descent moves every weight a small step against its gradient: the step is the learning rate η (eta) times the gradient. That works badly when the loss is shaped like a long, narrow valley, steep across and nearly flat along. The learning rate has to be small enough not to bounce out of the steep direction, so progress along the flat one is a crawl. The ratio of steepest to flattest curvature is the valley's condition number, and the linear models page shows gradient descent struggling on one. Two fixes are standard: momentum smooths the steps, and Adam gives each parameter its own step size.

Momentum keeps a running velocity. Each step it shrinks the old velocity by a factor β, usually 0.9, adds the new gradient, and moves the weights by η times the velocity. Gradients that keep pointing the same way pile up; gradients that flip sign every step mostly cancel.

Worked example

β = 0.9 and the gradient is 1 on every step. The velocity goes 1, 1.9, 2.71, … and levels off at 1/(1 − 0.9) = 10, so along a consistent direction momentum steps 10 times farther than plain gradient descent at the same η.

Now the gradient alternates +1, −1, +1, … The velocity settles at about ±0.53, so the zigzag is roughly halved.

That's why momentum speeds up the flat direction and calms the steep one. On a bowl it also raises the largest learning rate that doesn't diverge. If λ is the curvature of the steep direction, plain gradient descent blows up once ηλ passes 2; momentum holds until ηλ passes 2 + 2β, which is 3.8 at β = 0.9.[6]

Momentum still uses one learning rate for every parameter. When some parameters see much larger gradients than others, an η that's safe for the large ones is slow for the small ones. Adam divides each parameter's step by a running estimate of that parameter's own typical gradient size.[7] It keeps two running averages per parameter: m, a smoothed gradient like momentum's velocity, and v, a smoothed squared gradient. The step is η times m/√v.

Worked example

Two parameters with steady gradients, one of 1,000 and one of 0.001. Plain gradient descent at η = 0.001 moves them by 1 and by 0.000001.

Adam's averages settle at m = g and v = g², so m/√v = 1 for both, and both move by η = 0.001.

Here is the full update, run separately for every parameter at step t:
mt = β₁mt−1 + (1 − β₁)gt,   vt = β₂vt−1 + (1 − β₂)gt²,   θt = θt−1 − η m̂t / (√v̂t + ε)

In words: m is a smoothed gradient and v is a smoothed squared gradient. Each parameter moves by η times its smoothed gradient divided by its typical gradient size. The hats mark a start-up correction explained below, and ε stops a division by zero. The defaults β₁ = 0.9 and β₂ = 0.999 make v change much more slowly than m.[7]
Race three optimizers down a stretched bowl
Condition number
η for SGD and momentum
0.03
η for Adam
0.1
Step
Play
Reset
Loss ½(x² + 50y²) from (−2.5, 2.5), 0 steps. SGD at η = 0.03: loss 1.59e+2 (stable below 0.040). Momentum β = 0.9 at the same η: 1.59e+2 (stable below 0.076). Adam at η = 0.1: 1.59e+2; its per-coordinate normalization makes the step size nearly independent of the curvature.

Left: three optimizers start from the same point on a bowl shaded by loss, whose steep axis (y) is 50 times stiffer than the flat one (x). Right: their losses per step. Press Step (five updates per press) or Play, then drag the learning rates. At the defaults, plain gradient descent (SGD, purple) zigzags across the steep axis and reaches a loss near 10⁻⁵ after 200 steps; momentum with β = 0.9 (green) reaches about 5×10⁻⁸ at the same η; Adam at η = 0.1 (yellow) is near 10⁻⁸. Raise η past 0.04 and SGD diverges, while momentum keeps converging until just below 0.076, the (2 + 2β)/λ bound with λ = 50. Adam's path is almost a straight diagonal for its first steps: each coordinate moves by about η per step whatever its curvature, so η alone sets the step length.

Adam has a start-up problem. Both averages start at zero, so for the first steps they're far too small: with the default betas, after one step m holds only a tenth of the gradient and v only a thousandth of its square. Because v is the more shrunken of the two and the step divides by its square root, the uncorrected step comes out too large.[7]

Worked example

Constant gradient g = 1, default betas. After step 1: m = 0.1 × 1 = 0.1 and v = 0.001 × 1² = 0.001. The raw step is m/√v = 0.1/0.0316 ≈ 3.16 times η, instead of η.

The correction divides each average by the share of weight it has built up so far: m̂ = 0.1/(1 − 0.9¹) = 1 and v̂ = 0.001/(1 − 0.999¹) = 1, so the step is exactly η.

Without the correction the overshoot gets worse before it gets better. m catches up faster than v, so the raw step peaks at about 6.6η around step 12, and it's still about 1.26η at step 1,000.
Adam's moments warming up, with and without bias correction
β₁
0.9
β₂
0.999
Reset
Constant gradient g = 1, β₁ = 0.9, β₂ = 0.999. Step 1: m = 0.1000, v = 1.00e-3; raw m/√v = 3.162, corrected m̂/√v̂ = 1.000. Step 10: raw 6.528, corrected 1.000. Raw peak 6.57 at step 12; step 60: raw 4.14. Both averages start biased toward zero; v is biased more because β₂ is closer to 1, so the uncorrected ratio starts too large.

Left: Adam's step size in units of η over the first 60 steps of a constant gradient, uncorrected (purple) and bias-corrected (green). Right: the raw averages m and v climbing toward their true value of 1. Try lowering β₂. With a constant gradient of 1 the correct step is exactly η every time, and the corrected curve is flat at 1. The raw curve starts at (1 − β₁)/√(1 − β₂) = 3.16 for the defaults, climbs to 6.57 at step 12 while m catches up, and is still 4.14 at step 60; it decays toward 1 only as v catches up, which takes thousands of steps at β₂ = 0.999. Lower β₂ toward 0.9 and the bias disappears within a few dozen steps. nanoGPT uses β₂ = 0.95, whose average has a time constant of about 20 steps.[9]

An AdamW configuration also sets a weight decay. Weight decay pulls every weight a little toward zero on each step, which keeps the weights from growing large. AdamW applies it as a separate shrink of the weights before the Adam update, outside the moment averages, so Adam's division by √v doesn't rescale it.[8] At nanoGPT's peak learning rate of 6×10⁻⁴ and weight decay of 0.1, that shrink multiplies every decayed weight by 1 − 6×10⁻⁵ per step. AdamW is the variant nanoGPT and many other language-model trainers use.[9]
