# Overfitting and how to spot it

A photo classifier can score 100% on the photos it trained on and still mislabel the next photo a user uploads. A network with enough capacity can memorize its training set, noise included. Give 100 noisy points to a network with 1,000 weights and, trained long enough, it can bend its decision boundary around every one of them. On the training data it looks perfect; on new data it does worse than a simpler fit would.

This is overfitting: learning the noise in the training set as if it were a real pattern. The standard way to catch it is to hold out some of your data, never let the network train on it, and measure the loss on that held-out set during training too. When the training loss keeps falling but the held-out loss doesn't, the model has stopped learning things that carry over to new data.
Train / validation / test

The standard split is three sets. Training set: what the network sees and learns from. Validation set: held out, used to pick hyperparameters and decide when to stop training. Test set: held out even from you the engineer. Touch it once at the end to report final performance. Use it more than that and you're effectively training on it through the feedback loop.

The widget below trains a deliberately oversized network on 40 noisy points, jittered enough that a few land on the wrong side of the gap between the two classes. The solid green curve is loss on the training set. The dashed red curve is loss on a held-out validation set the network never sees. Validation loss usually bottoms out within the first 50 epochs; after that, training loss keeps falling while validation loss climbs.

Worked example

The widget's network is 2 → 16 → 16 → 2. Parameters: (2×16 + 16) + (16×16 + 16) + (16×2 + 2) = 48 + 272 + 34 = 354 weights and biases.

Against 40 training points, that's almost 9 adjustable numbers per point, plenty to fit each point's noise individually.
Live · Train vs. Validation
filled = training · outlined = validation · ◼ class 0 · ● class 1
training loss (solid) validation loss (dashed)
training set size 40
data noise 0.20
weight decay (regularization) 0.0000
TRAIN
Reset
epoch
0
train loss
···
val loss
···
Press TRAIN. The left panel shows the decision boundary with training points filled and validation points outlined; the right panel plots both losses per epoch. One network is scored on two sets of points. The decision boundary chases noise in the training set as the green solid curve drops further. The dashed red validation curve eventually starts climbing: from there on, the network is memorizing instead of generalizing. Drop the noise toward 0.02 and the classes stop overlapping, and the climb goes away.
How to recognize it

The diagnostic pattern is:

    Training loss keeps decreasing. The network is finding patterns in the training data.
    Validation loss decreases for a while, then turns around and starts increasing. Past this point, every additional epoch makes the network worse at the actual problem.
    The decision boundary develops weird bulges around individual training points, trying to capture each one's noisy position.

How to fix it

Four standard cures, often combined:

    More data. This is the cleanest fix: with a thousand points, the noise averages out and a network this size no longer has the capacity to memorize individual examples. Often not an option in practice.
    Smaller network. Fewer parameters means less capacity to memorize.
    Weight decay (L2 regularization). Penalize the network for having large weights, which pushes it toward smoother functions. Train with the weight-decay slider at 0.01 or above and the validation curve climbs far less than it does at 0.
    Early stopping. Track validation loss and keep a copy of the weights from its best epoch; once it has stopped improving for a while, stop and use that copy.

There are more techniques: dropout, which randomly zeros out neurons during training; batch normalization; and data augmentation. The four above are the baseline most training setups start from.
Check Your Understanding Q1 of 3

On the widget above, you crank the training set size down to 20 and the noise up to 0.30. You train without regularization. What pattern do you expect to see in the loss curves?
Both curves drop smoothly to zero, small data is fine.
Training loss drops steeply; validation loss reaches a minimum, then starts climbing.
Both curves stay flat, the network can’t learn from so few points.
0/3 correct
