# Checks before training a larger network
Matching shapes don't prove matching math

Two implementations can produce the same output shape while padding different sides, flipping the kernel, or ordering channels differently. Compare actual values on a small input that isn't symmetric. A symmetric image or kernel can hide all of these mismatches.

Build up the tests in order. Start with one patch whose products you can check by hand. Then use a single bright pixel away from the center, which shows whether outputs land in the right place. Then put nonzero values in more than one input channel to check the channel sum. For the backward pass, include weights used at several positions and input pixels covered by overlapping patches. Tests that only compare shapes catch none of these.

Check the input pipeline too. A correct model gives bad answers if inference feeds it BGR instead of RGB channel order, normalizes pixels differently, or resizes images differently from training. None of these is a convolution bug, which is why they're easy to miss. Keep preprocessing in one explicit place and compare one image end to end against a reference.

Split the data before you fit any preprocessing or choose an architecture. Neighboring frames from one video are nearly identical, so a random split by frame puts near-copies in both the training and test sets, and the test score can reward memorizing near-copies instead of generalizing. Split by the unit you want the model to generalize across, such as whole videos, scenes, sessions or people.
