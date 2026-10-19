# The inverse bind matrix

The modeler built the arm sticking straight out. To bend it, you first need each vertex's position measured from the joint that will move it, instead of from the model's origin. Mesh vertices are authored once, in model space, in the bind pose (the rest pose the mesh was modeled in, often a T-pose). The inverse bind matrix for joint j is the inverse of that joint's global transform in the bind pose. It maps a vertex from model space into joint j's local space, so the joint's animated transform can then carry it to its deformed position.

Worked example

In the bind pose the elbow sits at (0.3, 1.5, 0), and a forearm vertex at (0.4, 1.5, 0). The elbow's inverse bind matrix subtracts (0.3, 1.5, 0), leaving (0.1, 0, 0): 10 cm along the forearm. Now raise the arm as in §2, so the elbow is at (0, 1.8, 0) and turned 90°. The 10 cm offset turns into (0, 0.1, 0), and the vertex lands at (0, 1.9, 0), still 10 cm along the forearm. Skip the inverse bind and the animated elbow transform turns the whole model-space position instead: (0, 1.8, 0) + (−1.5, 0.4, 0) = (−1.5, 2.2, 0), a meter and a half off to the side.

The matrix that does that subtraction, for any joint, is the inverse of the joint's bind-pose transform:
eq. 2 · inverse bind matrix inverseBind[j] = inverse(globalBindTransform[j])

In words: before re-posing a vertex you have to cancel out the joint's rest pose, or the bind transform gets counted twice. Inverting the joint's bind-pose global transform gives the matrix that carries a model-space vertex into that joint's local space, ready for the animated transform to take over.
The classic omission, and glTF gives it to you

Forgetting the inverse bind (or applying it on the wrong side) makes the mesh explode the instant animation starts, because the animated joint transform gets applied to a vertex that's already in model space, double-counting the bind pose. glTF stores inverseBindMatrices directly (one per joint); use the stored values rather than recomputing them with a convention mismatch. The whole skinning idea is: undo the bind pose, then redo the current pose. Keep globalBindTransform (fixed) and globalPose (animated) clearly separate.
