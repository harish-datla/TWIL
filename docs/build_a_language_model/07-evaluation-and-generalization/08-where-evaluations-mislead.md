# Where evaluations mislead
Report every score with its baseline and its error bar

Report what a trivial predictor scores on the same split and metric, and the standard error of the score. A 92% that sits next to a 91% majority baseline with ±3 points of noise is a different finding from a 92% next to 50% with ±0.5.

Validation scores creep upward. Every experiment you run against the validation set and keep because it scored best nudges the score up, since the best of many noisy comparisons is biased high. The test set exists to catch that. Use it once, and if it has been used more than once, say so.

Perplexities rarely compare across papers. They're comparable only when the tokenizer, the evaluation text and the context protocol match. A lower number from a model with a different tokenizer, or one evaluated with a sliding window, may reflect the protocol rather than the model.[9]

Contamination doesn't show in the score. It comes from the training data. For a model trained on a web crawl, assume public test sets were in the crawl unless the training data was filtered against them, and prefer scores on held-out sets created after the training cutoff.[13]
