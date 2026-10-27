# Beyond SGD: smarter optimizers

Plain SGD has problems. In a long narrow valley it either zigzags between the walls (large η) or crawls along the nearly flat floor (small η). Near a saddle point it slows almost to a stop, and on a flat region it crawls. Real loss surfaces have all three. Most modern training uses SGD with momentum or an adaptive method such as Adam. Each keeps a small running memory of past gradients and uses it to shape the next step.
Momentum: a ball with mass

In a long valley, the gradient keeps pointing the same way along the floor but flips sign from step to step across it. Momentum uses that difference. Plain SGD is a marble that stops the instant you stop pushing it; momentum SGD is a ball with mass. It keeps a running blend of recent gradients, picks up speed where they agree, and cancels out where they keep flipping.

Worked example

Blend factor β = 0.9, learning rate η = 0.1, velocity v starting at 0. Each step, v becomes 0.9 v − 0.1 g.

Gradient g = 1 every step (the valley floor): v goes −0.1, −0.19, −0.271, and keeps growing toward −1, ten times the plain SGD step of −0.1.

Gradient flipping +1, −1, +1, −1 (across the valley): v goes −0.1, 0.01, −0.091, 0.018. It stays small, so the zigzag is damped.

The update keeps a velocity v and moves the weights by it instead of by the raw gradient:
Momentum SGD v ← β v − η · ∇C w ← w + v

In words: each step, keep a fraction β (typically 0.9) of the previous velocity and add a fresh step of −η times the current gradient. Then move the weights by that velocity. Directions where the gradient agrees build up speed; directions where it alternates cancel.
RMSProp: adapt the step size per parameter

Different parameters can have wildly different gradient scales. A weight in the output layer might routinely see gradients of size 1.0; a weight five layers deep might see 0.001. Using the same learning rate for both is a compromise neither one wants. RMSProp[12] tracks a running average of the squared gradient for each parameter and divides each update by the square root of that average. Big-gradient parameters get smaller steps; small-gradient parameters get bigger ones.

Worked example

Two weights. One keeps seeing gradients of about 1.0, the other about 0.001. After many steps, the running averages of their squared gradients settle near 1.0 and 0.000001, whose square roots are 1.0 and 0.001.

Steps: η × 1.0 / 1.0 = η, and η × 0.001 / 0.001 = η. Both weights now move about the same distance per step, where plain SGD would move the second one a thousand times less.

The update keeps a running average s of each parameter's squared gradient and divides the step by its square root:
RMSProp s ← ρ s + (1 − ρ) · (∇C)2 w ← w − η · ∇Cs + ε

In words: keep a running average of each parameter's squared gradient, an exponential moving average in which each step keeps a fraction ρ of the old average and mixes in 1 − ρ of the new value, so recent steps count most. Then divide each update by the square root of that average. The tiny ε (typically 10−8) prevents division by zero when a parameter's gradients have all been zero so far.
Adam: momentum + RMSProp + bias correction

Adam[13] combines the previous two ideas. It keeps a momentum-style running average of the gradient, m, and an RMSProp-style running average of the squared gradient, v. Both start at zero, so for the first steps they're too small. Adam divides each by a correction factor that undoes this, a step called bias correction. Adam and its variant AdamW are the most common default optimizers in deep learning today.

Worked example

First step, default β1 = 0.9 and β2 = 0.999, gradient g = 2.

m = 0.1 × 2 = 0.2 and v = 0.001 × 4 = 0.004. Uncorrected, the step would be η × 0.2 / √0.004 ≈ 3.16 η.

Corrected: m̂ = 0.2 / (1 − 0.9) = 2 and v̂ = 0.004 / (1 − 0.999) = 4, so the step is η × 2 / √4 = η.

The full update, where t counts the steps taken so far:
Adam m ← β1 m + (1 − β1) · ∇C
v ← β2 v + (1 − β2) · (∇C)2
m̂ = m1 − β1t, v̂ = v1 − β2t
w ← w − η · m̂v̂ + ε

In words: track how the gradient has been pointing on average (m) and how big it has been on average (v). Divide each by 1 − βt to undo their start at zero; that factor approaches 1 as training goes on. Then take a momentum-style step, divided per parameter by the typical gradient size.

Racing the four optimizers down the same surface shows the differences. On the Rosenbrock function (a banana-shaped valley that is a standard optimizer stress test; the widget uses a rescaled version) at the default η = 0.05, plain SGD has covered only the first stretch of the curve after 320 steps, Momentum and Adam zigzag across the valley before settling onto its floor, and RMSProp follows the floor from the start and gets closest to the minimum. Toy surfaces like these show how each update rule behaves; they don't rank optimizers on real networks.
Live · Optimizer Race
surface
learning rate η 0.050
Race ▶
Reset ⟲
SGD Momentum RMSProp Adam step 0
Press Race to start four optimizers from the same point: red is SGD, yellow Momentum, green RMSProp, purple Adam. Each one turns the gradient at its current position into a step in its own way, so the paths split apart. Try the other surfaces and learning rates. The Rosenbrock function is a common stress test: a narrow curved valley with a single minimum at (1, 1).
Check Your Understanding Q1 of 2

You set Adam’s β1 = 0 and β2 = 0. What optimizer have you accidentally recreated?
Sign-SGD: each coordinate moves by ±η, regardless of the gradient’s magnitude.
Pure momentum SGD.
RMSProp.
0/2 correct
