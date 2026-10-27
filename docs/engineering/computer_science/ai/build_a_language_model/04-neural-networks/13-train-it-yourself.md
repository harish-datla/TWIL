# Train it yourself

The widget below runs a compact copy of the JavaScript above (same algorithm, same initialization, shorter variable names) in your browser. Pick a dataset, tweak the architecture, hit TRAIN. The left panel shows the predicted class probability over the input space, with the white line marking the decision boundary where both classes are equally likely; the chart under the controls plots the average training loss per epoch.
Live · Neural Network Trainer click on the left canvas to add points · shift-click for class 0
INPUT SPACE · decision boundary forms as the network learns
dataset
activation
hidden units / layer 8
learning rate η 0.030
data noise 0.10
TRAIN
Step 1
New data
Reset
epoch
0
loss
0.7171
accuracy
50.0%
throughput
···
Pick a dataset and press TRAIN, then watch the white boundary bend around the points while the loss chart falls and the accuracy climbs. Click the left canvas to drop a class-1 point, or shift-click for class 0, and the network adapts. Class 0 (blue) draws as squares and class 1 (purple) as circles, so the two are distinguishable without color. Architecture: 2 → hidden → hidden → 2, trained with SGD at batch size 1 and cross-entropy loss.
Things to try

    Push the learning rate up to 0.2 on the spiral. The loss curve spikes instead of settling, and the accuracy usually ends well below what η = 0.03 reaches. With ReLU, the oversized steps also tend to push several hidden units permanently negative: the dying-ReLU failure from §10.
    Drop the learning rate to 0.001. The loss falls smoothly, with few upticks, but takes several hundred epochs to get where η = 0.03 gets in about fifty.
    Use only 2 hidden units on the spiral. The network doesn't have the capacity to bend the boundary enough and stalls around 70 to 75% accuracy. At 8 or 16 units it fits the curve.
    Switch from ReLU to sigmoid on the spiral. The loss sits on a plateau for much longer before it starts to fall. With only two hidden layers this is a mild version of the vanishing-gradient problem from §10: every σ′ factor is at most 0.25. Tanh, whose slope reaches 1, trains much closer to ReLU's pace.
    Try XOR with 2 hidden units and sigmoid. The loss usually sits near 0.69 (chance level for two classes) for dozens of epochs, sometimes over a hundred, then drops as the hidden units find the two cuts XOR needs. A single perceptron, with no hidden layer, can never do this: Minsky and Papert's point. With ReLU and only 2 units, runs often stall with dead units.

