# Where linear models go wrong
Standardize with training statistics only

Compute the mean and standard deviation used for scaling on the training split only, then apply those same numbers unchanged to validation and test data. Computing them on the whole dataset leaks information about the test set into the fit. Computing them separately for each split makes the same raw input mean different things in different splits.

Report held-out loss next to training loss. A widening gap between the two is the signal to regularize or simplify the model rather than train longer.[5]

Near-duplicate features have meaningless individual weights. Take two features that are nearly proportional, such as floor area and number of rooms in a building whose rooms are all about the same size. The fit can trade weight between them almost freely, and small changes in the data swing both. Only their combined effect is pinned down.[6]

Scaling changes the safe learning rate. A learning rate that works on standardized inputs can diverge on raw ones, because the curvature grows with the square of a feature's scale. When the loss shoots toward infinity in the first few steps, check the input scaling first.
