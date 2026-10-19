# Where the shapes lie
A product that runs is not a product that is right

A weight used in the wrong orientation, a transpose forgotten when porting between libraries, or a broadcast that stretched the wrong axis all produce numbers of the right shape. Check a layer against a 2×2 case you worked out by hand before trusting it on a 4096×4096 one.

Cosine similarity breaks near zero. It divides by both lengths, so it's undefined for an all-zero vector, and for a nearly-zero one a tiny change can swing the angle anywhere. Check that embedding lengths stay well away from zero before normalizing them, and keep the raw dot product when length carries meaning.

Floating-point dot products aren't exact. Adding up a long vector piles up rounding error, and the error depends on the order of the additions, so two correct matrix kernels can disagree in the last few digits. The floating point page explains why. In practice, compare results with a tolerance scaled to the size of the numbers, as the programs above do.

Determinants and eigenvalues need square matrices. For a rectangular weight matrix, the singular values tell you how much a layer can stretch its input; the largest one is called the spectral norm.[1] Weight matrices with large spectral norms are one common contributor to activations that explode in deep stacks, which the training page returns to.
