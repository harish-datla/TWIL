# The skeleton

Bend the elbow and the forearm has to follow it, the hand has to follow the forearm, and the fingers have to follow the hand. A skeleton is a tree of joints, each storing a local transform (translation, rotation, scale) relative to its parent. A joint's model-space (global) transform is its parent's global transform times its local, walking from the root: the same scene-graph propagation as in Going 3D[1].

Worked example

A shoulder sits at (0, 1.5, 0) m, and the elbow's local transform puts it 0.3 m along the shoulder's x axis. Rotate the shoulder 90° about z and the elbow's global position becomes the shoulder's position plus that rotated offset: (0, 1.5, 0) + (0, 0.3, 0) = (0, 1.8, 0). The arm now points straight up, and nobody touched the elbow's local transform.

The rule that produced that position, for every joint:
eq. 1 · joint world transform globalTransform[joint] = globalTransform[parent] · localTransform[joint]

In words: a joint's place in the model is its parent's place times its own local transform. Walk the skeleton parent-first and each joint just multiplies onto its parent, which is why the root has to be processed before its children. Hover any symbol to see what it stands for.
"Joint" vs "bone", and parent-before-child

glTF (Khronos's open 3D asset format) and most math call the node a joint; a "bone" loosely means the visible segment between a joint and its parent. The transform lives on the joint. And a parent's global transform must be computed before its children's: walk the node tree (or sort so parents precede children) rather than iterating the joint array in storage order, or a child reads a stale parent. The skeleton can have any number of joints; that's separate from the per-vertex influence cap (§4).
