# Warmup, decay and clipping

A fixed learning rate is wrong at both ends of a run. At the start, the weights are random and the gradients point in poor directions, so a large step can send the loss up instead of down. Near the end, each step's gradient comes from a minibatch, a small random sample of the data, so it's noisy. A large η keeps the weights bouncing around the minimum instead of settling into it.

Language-model trainers commonly handle both ends with a schedule. A warmup ramps η up in a straight line from near zero over the first steps, which limits early divergence in deep networks.[10] A cosine decay then lowers it along half a cosine wave: η stays near the peak for a while, then falls smoothly to a floor. Loshchilov and Hutter proposed the cosine shape.[10] nanoGPT's schedule is exactly these two pieces, with a floor of a tenth of the peak.[9]

Worked example

nanoGPT's defaults: peak 6×10⁻⁴, 2,000 warmup steps, cosine decay until step 600,000, floor 6×10⁻⁵.[9]

At step 1,000, halfway through warmup, η ≈ 3.0×10⁻⁴.

During the decay, η = floor + ½ × (peak − floor) × (1 + cos(π × fraction of the decay done)). Halfway through, the cosine is 0, so η = 6×10⁻⁵ + ½ × 5.4×10⁻⁴ = 3.3×10⁻⁴.

From step 600,000 on, η stays at the floor, 6×10⁻⁵.
Shape a warmup-plus-cosine schedule and run noisy SGD under it
Warmup steps
20
Total steps
200
Floor, fraction of peak
Gradient noise σ
Reset
Peak η = 0.036, warmup 20 steps, cosine decay to 0.0036 at step 200. η at step 1: 0.00171; at the end of warmup: 0.03600; halfway through decay: 0.02008. SGD with gradient noise σ = 1 on the ½(x² + 50y²) bowl after 200 steps: constant η → loss 3.28e-2, scheduled → 1.71e-3.

Left: the learning rate at each step. Right: the loss of plain gradient descent on the stretched bowl from section 3, run once at a constant η equal to the schedule's peak (purple) and once under the schedule (green). Change the warmup, length and floor, then set the noise to 0. Seeded random noise is added to every gradient to stand in for minibatch sampling. With noise σ = 1, the constant-rate run flattens out near 3×10⁻², because each step kicks the weights by about η times the noise, while the scheduled run keeps falling as η shrinks and ends near 2×10⁻³. At noise 0 the result flips: on a noise-free bowl the constant rate reaches 10⁻⁶ and the decayed one stalls near 10⁻³, because decay only slows a clean descent.

Even with a schedule, an occasional batch produces a huge gradient, and a single huge step can throw the weights far from where training had brought them. Gradient clipping is the safety rail. PyTorch's clip_grad_norm_ treats the gradients of all parameters as one long vector and measures its length, its norm. If the norm exceeds a chosen maximum, every gradient is scaled down by the same factor, so the update keeps its direction and only its length is capped.[11] nanoGPT clips at 1.0 on every step.[9]

Worked example

Two parameters with gradients (3, 4). Their combined length is √(3² + 4²) = 5. With a maximum of 1, both are scaled by 1/5 to (0.6, 0.8): same direction, length 1.
