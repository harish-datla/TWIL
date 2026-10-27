# Steer generation with guidance

A model that generates from a prompt has to make what the prompt asks for. So the network also takes a condition, such as a class label or an encoded text prompt. Classifier-free guidance makes it follow the condition more strongly. At each step it computes two noise predictions, one with the condition and one without, and pushes further in the direction where they differ. In Ho and Salimans' method, training drops the condition on a random fraction of examples, so one network learns to make both predictions.[5]

Worked example

In the guidance widget, the condition means "from the +2 cluster". At x_t = −1 and ᾱ = 0.4, the unconditional prediction is ε̂ ≈ 0.25 and the conditional one is ε̂ ≈ −2.64.

With guidance scale g = 2: 0.25 + 2 × (−2.64 − 0.25) = 0.25 − 5.78 = −5.53.

The guided prediction overshoots the conditional one, so the step pushes the sample harder toward +2.

The combination is one line.
ε̂_guided = ε̂_unconditional + g × (ε̂_conditional − ε̂_unconditional)

In words: start from the unconditional prediction and move toward the conditional one by g times the gap between them. g = 0 ignores the condition, g = 1 gives the plain conditional prediction, and g above 1 goes past it. Ho and Salimans write the scale as w, where g = 1 + w.[5]

"Classifier-free" means no separately trained classifier network is needed to steer the sampler, unlike the earlier classifier-guidance method. The condition itself is still there. In the widget, the unconditional data is the two-cluster mixture and the condition picks the +2 cluster. Both predictions are exact, so you can see exactly what combining them does.
Combine conditional and unconditional scores
Guidance scale g
2
Signal power ᾱ
0.4
Reset
At x_t=−1: ε̂_uncond=0.2451, ε̂_cond=-2.6422, guided ε̂=-5.5294. Scale 0=unconditional, 1=conditional, >1=extrapolation toward the +2 condition.

You're looking at the score, the direction toward more likely data, for the unconditional and conditional models (left), and the guided score (right). Drag the guidance scale and ᾱ. At the default scale of 2, the readout gives the matching noise predictions at x_t = −1: 0.2451 without the condition, −2.6422 with it, and −5.5294 guided. The best noise guess is −√(1 − ᾱ) times the score, so the same blend applies to either. A high scale is an extrapolation, not a probability, and doesn't guarantee better output.
What distribution does guidance aim at?

Raising the scale makes samples follow the condition more closely and reduces their variety. Ho and Salimans measure this as a trade-off between sample quality and diversity, similar to the one classifier guidance gives.[5] The best setting depends on the model and the task, so measure it.
Check Your Understanding Q1 of 2

What is random in an η=0 DDIM run?
Every intermediate reverse update.
Nothing, so all generated samples are identical.
The initial draw; subsequent updates are deterministic given the predictor.
0/2 correct
