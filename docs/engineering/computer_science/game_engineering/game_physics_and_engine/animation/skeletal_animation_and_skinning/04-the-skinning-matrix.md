# The skinning matrix

A vertex in the crease of the elbow belongs partly to the upper arm and partly to the forearm; give it to either one alone and the skin tears or folds at the joint. Each vertex is bound to up to N joints (glTF: 4 per set) with weights that sum to 1, each weight saying how much that joint pulls on the vertex. The skinning matrix is the weighted sum of each influencing joint's skinning transform = jointGlobalAnimated[j] · inverseBind[j].

Worked example

Keep the shoulder still and bend only the elbow 90°. A vertex just past the elbow, at (0.35, 1.5, 0), has weight 0.5 on each joint. The upper arm hasn't moved, so its skinning matrix leaves the vertex at (0.35, 1.5, 0). The forearm's turns the vertex's 5 cm offset from the elbow upward, to (0.3, 1.55, 0). The blend is the average, (0.325, 1.525, 0): the crease rounds off between the two bones. Halve both weights so they sum to 0.5 and the result is (0.1625, 0.7625, 0), pulled halfway to the model's origin.

Written out, the weighted blend for one vertex is:
eq. 3 · linear blend skinning skinned = ( Σi weighti · jointGlobalAnimated[jointi] · inverseBind[jointi] ) · position

In words: for every joint that influences the vertex, undo that joint's rest pose (inverseBind), apply where the joint is now (jointGlobalAnimated), and scale by its weight. Σi sums those weighted matrices, and the blend transforms the vertex. Because the weights sum to 1, a vertex at the knee bends smoothly between thigh and shin instead of tearing.
Order, weights, and the normal

The order is jointGlobalAnimated · inverseBind: the inverse bind on the right (applied first, to a model-space vertex), the animated global on the left. Weights must be non-negative and sum to 1: weights summing to s scale the skinned position by s about the model-space origin. 4 influences is glTF's per-set cap, not a universal law; more needs a second JOINTS_1/WEIGHTS_1 set[2]. And transform the normal too, by the skinning matrix's inverse-transpose under non-uniform scale (the 3D Math rule); the cheap normalize(mat3(skin) · normal) path is exact only when the joints carry no non-uniform scale.

The widget skins a two-joint limb with exactly that blend:
play
show weights
weights sum to 1
weights sum to 0.5
A two-joint limb bending at the elbow, with its skin colored by which joint each vertex follows. Untick play to freeze it, then switch to "weights sum to 0.5". The limb bends; the skin follows because each vertex blends its two joints. The color shows the weight falloff straddling the elbow. Switch to "weights sum to 0.5" and the skin shrinks toward the model-space origin (here at the shoulder) and pulls away from the bones: LBS computes Σ wi·(Mi·p), so weights summing to 0.5 halve every skinned position. That is why weights must sum to 1.
The CPU matrix palette (walk the hierarchy, then build skinning matrices)
C++ · glm
Rust · glam

// joints sorted so a parent always precedes its children
for (size_t joint = 0; joint < jointCount; ++joint) {
    int parent = parentIndex[joint];                         // -1 for the root
    glm::mat4 local = localPose[joint].toMatrix();           // animated TRS this frame
    globalPose[joint] = (parent < 0) ? local
                                     : globalPose[parent] * local;   // parent.global * local
}
for (size_t joint = 0; joint < jointCount; ++joint)
    skinningMatrix[joint] = globalPose[joint] * inverseBind[joint];  // ORDER: animated * invBind

