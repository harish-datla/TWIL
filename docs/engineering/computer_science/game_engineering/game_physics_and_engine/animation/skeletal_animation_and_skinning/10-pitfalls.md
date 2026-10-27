# Pitfalls
Missing the inverse bind
Mesh explodes. skinningMatrix = jointGlobalAnimated · inverseBind, in that order.
Weights not summing to 1
Mesh shrinks/swells. Normalize; keep them non-negative.
Parent after child
Stale parent global. Walk the tree or sort parents-first.
Lerping euler/matrix for rotation
Wrong arc / shrinking. Slerp or nlerp quaternions.
Blending global matrices
Non-rigid in-betweens. Blend local TRS, then recompute globals.
"DQS is strictly better"
It bulges on bend and can't scale/shear. LBS + helper joints is the default.
JOINTS_0 indexing nodes
It indexes skin.joints, not the node array. Tangled mesh.
mat3(skin)·normal under scale
Rigid-only shortcut. Use the inverse-transpose for non-uniform scale.
