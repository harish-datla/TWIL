# Three splits and the leak between them

Say you train a spam filter and it gets 99% of its training emails right. That number tells you little about the emails it will see next week, because the model may have memorized quirks of the training set. What you care about is the generalization error: how often it's wrong on new examples from the same source. You estimate it on examples you held back from training.[1]

In practice the data is split three ways, one per job. The training set is what the model learns from. The validation set is what you check while developing, to choose between models and settings such as the learning rate. The test set is used once, at the end. Every decision you make by looking at a set's score tunes the model a little toward that set, so a set you've used for decisions gives an optimistic score.[1][2]

Leakage is any route by which information from the held-out data reaches training. The obvious one is tuning on the test set. A subtler one is preprocessing. A scaler, a feature selector or a vocabulary that's fit on the full dataset before the split has already seen the test examples, and the model inherits that knowledge.[2]

scikit-learn's guide shows how bad this gets with data that's pure noise: random features and random labels, where no model can beat chance. Picking the features most correlated with the labels on all the data, and only then splitting, gives a test accuracy far above chance.[2]

Worked example

100 examples with coin-flip labels and 1,000 random features. With 100 examples, a pure-noise feature's correlation with the labels typically lands within about ±0.1 of zero (its standard deviation is roughly 1/√100).

Check 1,000 such features and the strongest one, by chance alone, is typically around 0.33. If those chance correlations were picked using the test examples too, they hold up on the test examples, and the score looks like skill.
Select features before or after the split, on pure noise
Random features
1000
Features kept
10
Seed
1
Reset
100 examples with random labels and 1000 random features; keep the 10 features most correlated with the label, then classify the held-out half by nearest centroid. Selecting on all 100 examples first: held-out accuracy 86%. Selecting on the training half only: 44%. The labels are coin flips, so anything far from 50% is the leak, not skill.

The bars compare held-out accuracy when the features are selected on all 100 examples versus on the training half only; chance is 50%. Try more random features, fewer kept features, and other seeds. Every example has a coin-flip label and random Gaussian features, so the true accuracy of any classifier is 50%. Choosing the ten features most correlated with the label on all hundred examples, then testing on a held-out half, scores in the 70s and 80s: the selector found features that happen to match the held-out labels by chance. Selecting on the training half only lands near 50%. More candidate features make the leak worse, because there are more chance correlations to find.

When data is scarce, holding out a big validation set wastes examples. k-fold cross-validation splits the data into k parts (folds), trains k times with a different fold held out each time, and averages the k validation scores, so every example is used for validation once. When some classes are rare, stratified folds keep each class's share the same in every fold, so no fold ends up with none of a class.[3] The leakage rule still applies inside each fold: fit every preprocessing step on that fold's training part alone.[2]
