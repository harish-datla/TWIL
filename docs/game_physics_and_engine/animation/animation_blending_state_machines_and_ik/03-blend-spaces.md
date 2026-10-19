# Blend spaces

The character can move at any speed between a walk and a sprint, and in any direction, but you only have a handful of clips. A state's pose is often a blend tree, which mixes several clips by runtime parameters, rather than a single clip. A 1D blend places a parameter (speed) on a line of sorted samples (idle, walk, run); find the two bracketing samples and blend by the normalized fraction. A 2D blend mixes by two parameters (e.g. forward speed × strafe speed) and needs a real 2D weighting scheme[1][3].

Worked example

A 1D speed blend has idle at 0, walk at 1.4 and run at 4.0 m/s. At 2.0 m/s the bracketing samples are walk and run, and the fraction is (2.0 − 1.4) ÷ (4.0 − 1.4) = 0.23, so the pose is 77% walk and 23% run.
2D blending is not two 1D blends

With the usual locomotion samples, which don't fill a grid (idle in the middle, one clip per direction), a 2D blend can't be built from two independent 1D blends: mixing an X blend with a Y blend is exact on the axes, but off them (a diagonal strafe) it keeps too much idle and bends the motion toward the nearer axis. Unity's freeform 2D blend types use Gradient Band Interpolation (Johansen): each sample's influence is min over j of [1 − (p−p_i)·(p_j−p_i) / |p_j−p_i|²], clamped at zero and normalized. It needs no triangulation, and each sample's influence reaches exactly to its neighbors however unevenly they're spaced[8]. Barycentric (triangulation) is a valid alternative but needs a mesh of the samples and extrapolates badly outside the hull. Directional locomotion uses Johansen's polar variant (angle + magnitude), because in Cartesian space a walk sample's influence can reach out to speeds beyond the run samples in that direction[8]. Whichever scheme, it only computes weights; the blend itself is still the local-space per-joint one.

Worked example

Take the widget's five samples (idle at the center, forward, back, left and right one unit out) and a query at (0.6, 0.3), right and slightly back. For the right sample, the band against idle is 1 − 0.4 = 0.6, and its bands against the other samples are larger, so its raw weight is 0.6. Idle comes out at 0.4 and back at 0.3; forward and left come out at 0. Normalized, that's 46% right, 31% idle and 23% back, and the blended motion points exactly where the query does. Two 1D blends give 40% right, 50% idle and 10% back, and their motion points 13° off.

The widget computes both schemes for a point you can drag:
gradient band
two 1D blends
Five sample clips, each drawn as a disc sized by its weight, and a query point that orbits on its own until you drag it. Drag it off the axes, then switch to two 1D blends. The gradient-band weights and the blended motion (solid arrow) update live against the requested motion (dashed). Switch to two 1D blends and, off the axes, the solid arrow bends toward the nearer axis and idle keeps too much weight: at (0.6, 0.3) it is off by 13°. Neither scheme reaches the corners with only these five samples; that takes diagonal clips.
