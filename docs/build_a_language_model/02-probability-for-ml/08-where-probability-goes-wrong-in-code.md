# Where probability goes wrong in code
Never take the log of a probability you computed by exponentiating

exp(z) for a large negative z underflows to zero, and log(0) is −∞, so the loss becomes infinite or NaN. Work in log space from the logits with log-sum-exp, as the programs above do, and the problem never comes up.

The two directions of KL divergence answer different questions, and papers say which one they minimize because the fitted result differs. When a divergence shows up in a loss, check which argument is the data and which is the model before drawing any conclusion from it.

A softmax output always adds up to 1, but that doesn't make its numbers trustworthy. A model can report 0.95 for predictions that are right 70% of the time; a model whose stated confidence matches how often it's right is called calibrated. Cross-entropy training pushes toward calibration on the training distribution, and the evaluation page measures how far a model drifts from it on held-out data.

Likelihood and probability are read in opposite directions. Fix θ and add up P(data | θ) over every possible dataset, and you get 1. Fix the data and add it up over every θ, and you can get any number. So a likelihood is not a distribution over parameters. Getting one takes a prior and Bayes' rule from section 1, which is exactly what the posterior is for.
