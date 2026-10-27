# The best guess can be a value no example has

Hand a model a noisy value and ask for the clean one. Often several clean values could have produced it: a noisy 0 in the toy dataset could have come from the −2 cluster or the +2 cluster, equally likely. A model trained with squared error learns to output the average of the possibilities, each weighted by how likely it is. That's the posterior mean. Squared error rewards it because it gives the smallest expected squared miss, even when the average itself is a value that almost never occurs in the data.

Worked example

Noisy value x_t = 0 at ᾱ = 0.4. The two clusters are equally likely sources.

If it came from the −2 cluster, the best guess for the clean value is about −1.81. From the +2 cluster, about +1.81. The posterior mean averages them: 0, in the gap where almost no clean data lies.

Why squared error picks 0: if the truth is −1.81 or +1.81 with equal odds, guessing 0 costs 1.81² ≈ 3.3 on average. Guessing +1.81 costs nothing half the time and 3.61² ≈ 13.0 the other half, so about 6.5 on average.

For one cluster, the best guess has a closed form. With s = √ᾱ for the signal weight and σ² = 1 − ᾱ for the noise variance, it starts at the cluster's center and moves toward what the observation suggests.
E[x₀ | x_t, component] = μ + (s τ² / (s²τ² + σ²)) × (x_t − s μ)

In words: start from the cluster's center μ, then correct it by how far the observation x_t lands from where that center would sit after noise, s μ. The correction is bigger when the cluster's own spread τ² is large compared with the noise σ²: the less noise, the more the observation is trusted. For the two-cluster mixture, compute this for each cluster and average the two, weighting each by its Bayes responsibility: how likely that cluster is to have produced x_t.
Where the formula comes from

Compare posterior modes with their mean
Noisy observation
0
Signal power ᾱ
0.4
Reset
Observed x_t=0; component posterior weights [0.500, 0.500]; E[x₀|x_t]=0.0000; E[ε|x_t]=0.0000. The mean need not sit at a density peak.

You're looking at the density of noisy observations with your chosen observation marked (left), and the exact density of the clean value given that observation, with its mean marked (right). Drag the observation and the noise level. At the defaults (observation 0, ᾱ = 0.4) the right panel has two humps near ±1.8 and the mean sits at 0 between them. Move the observation toward ±2 and one hump takes over, pulling the mean onto it. Every curve here is exact arithmetic for the known mixture.

The best noise guess follows from the best clean guess by rearranging the recipe: E[ε | x_t] = (x_t − s E[x₀ | x_t]) / σ, for σ > 0. It's also tied to the score of the noisy data, the slope of its log-density, which points toward where noisy data is more likely: E[ε | x_t] = −σ ∇log p_t(x_t). So predicting the noise and pointing uphill toward likely data are the same job. Song and Ermon built a generator directly on scores of noisy data.[3] This page computes the score exactly from the known mixture.

That's also why the averaging isn't fatal. A diffusion sampler doesn't take one posterior mean of a heavily noised sample and stop. It takes a step in the predicted direction, predicts again at a lower noise level, and repeats. At low noise the observation pins down which cluster it came from, and the best guess stops sitting between them.
Check Your Understanding Q1 of 2

A symmetric two-mode posterior has mean zero. Is zero necessarily its most likely clean value?
Yes; a mean is the location of the highest density.
Only if the added noise has a nonzero mean.
No; the mean can lie between modes.
0/2 correct
