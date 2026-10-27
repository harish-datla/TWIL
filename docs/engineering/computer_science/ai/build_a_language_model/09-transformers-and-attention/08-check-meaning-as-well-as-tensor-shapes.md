# Check meaning as well as tensor shapes
An attention map is not an explanation of correctness

Attention weights describe one internal mixing step. A confident-looking map doesn't show that the output is factually right, that the model's uncertainty is calibrated, or why the final model picked a token.

Test a kernel with small, lopsided vectors. If all the keys are equal, every weight is the same, and a transposed matrix or a broken mask gives the same answer as a correct one. Change a forbidden future value and assert that earlier outputs don't move. Compare cached and uncached results for several prompt lengths, for chunks of more than one token, and for nonzero position offsets.

Check the mask convention every time you call a library. One API treats true as "allowed" and another treats it as "masked", so a correct boolean array with the opposite meaning silently computes a different model. Read the library's contract and check one case by hand.

Correct arithmetic isn't the same as useful predictions. These widgets compute exact attention on small arrays whose values were never trained on language. Building the layer gets the numbers right; useful predictions need suitable data, a training objective and an evaluation on the tasks you care about.
