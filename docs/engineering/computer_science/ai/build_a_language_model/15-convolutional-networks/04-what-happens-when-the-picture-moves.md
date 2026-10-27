# What happens when the picture moves

A cat classifier should say "cat" whether the cat sits in the middle of the photo or a few pixels to the left. Convolution gets part of the way there. Slide the input one pixel and a stride-1 convolution's output slides one pixel too, with the same values, away from the borders. That property is translation equivariance: the output moves with the input. A classifier needs something stronger, an answer that doesn't change at all when the input moves. That's called invariance. A CNN, a network built from convolution layers, moves toward invariance by discarding position in stages, and those stages are where shifts cause trouble.

The usual way to discard position is downsampling: keep fewer positions. Max pooling with a 2-wide window and stride 2 splits a row into pairs and keeps the larger value of each pair.[4] A stride-2 convolution also produces outputs at only every second position. In both cases a one-pixel shift can move a value from one window into the next, which changes the output instead of just sliding it.[5]

Worked example

Take the row (0, 0, 1, 0.3, 0, 0, 0, 0). Max pooling over pairs gives (0, 1, 0, 0): the 1 and the 0.3 share the second pair.

Shift right by one: (0, 0, 0, 1, 0.3, 0, 0, 0). The 1 is still in the second pair, but the 0.3 has moved into the third, so the output is (0, 1, 0.3, 0). The shift added a response instead of moving the old one.

Shift by two instead and the output is (0, 0, 1, 0): the original output moved one place, a clean shift.

Average pooling keeps the total, 0.65 before and after the one-pixel shift, but splits it across two outputs: (0, 0.65, 0, 0) becomes (0, 0.5, 0.15, 0).

The widget runs this example. It uses circular shifts, where values pushed off the right end come back on the left, so the border never interferes and every change comes from the pooling windows.
Move a peak across pooling boundaries
Circular shift
1
Pooling
Reset
Circular input shift 1; stride-two max pooling. Base [0, 1, 0, 0]; shifted [0, 1, 0.3, 0]; same-index L1 change 0.30.

You're looking at the input row before (purple) and after (green) the shift, and the pooled output of each. Drag the shift and switch between maximum and average. At the default shift of 1, max pooling gains a 0.3 in a new slot and the readout shows an L1 change of 0.30. L1 compares outputs at the same index, so it counts any movement as change, even the clean shift you get from an even input shift; it isn't a measure of broken equivariance. A shift of two input positions moves the pooled output by one.

Signal processing calls this aliasing: sampling too sparsely makes the result depend on exactly where the samples fall. The standard fix is to blur slightly before downsampling, so no detail is finer than the samples can capture. Zhang (2019) added that blur to the downsampling layers of standard CNNs and reported outputs that changed less under small shifts, along with higher ImageNet accuracy across several common architectures.[5] The blur costs some fine detail, and it reduces shift sensitivity rather than removing it.

At the very end, a classifier needs one answer for the whole image. Global average pooling gets there by averaging each channel's whole map into one number, which removes position completely. A detector that has to say where the cat is can't do that, because position is exactly what it must report.
