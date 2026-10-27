# Where autograd bites
Zero the gradients, every step

PyTorch-style engines add into a parameter's gradient buffer on every backward call, so the buffer still holds the previous step's gradient unless you clear it. Forget, and every past batch's gradient gets added into the current update. The loss often still falls for a while, which hides the bug.

Don't overwrite what the backward pass needs. Changing in place a value the backward pass will read produces a wrong gradient or an error. PyTorch keeps a version counter on saved tensors and raises an error if one was overwritten, and it discourages in-place operations in most code for this reason.[3]

Recomputation has to repeat the forward pass exactly. A global flag that changed, or a dropout mask drawn from a different random state, makes the recomputed segment differ from the original. The result is an error or a silently wrong gradient. PyTorch's checkpoint saves and restores the random-number state by default for this reason.[4]

Gradient checks are for testing derivative rules. A centered check costs two loss evaluations per parameter, far too slow to train with. Run it on a small model, at ordinary points away from corners, with h near 10⁻⁵ in double precision, as on the linear models page. In single precision the rounding floor is much higher, so the check has to be correspondingly looser.
