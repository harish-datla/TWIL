# The animation clip

An animator keys the arm at 0.0 s and 0.5 s, and the game needs a pose every 16.7 ms in between. A clip is a set of per-joint keyframe tracks, separate tracks for translation, rotation, and scale (glTF animation channels target exactly these three node properties, plus morph-target weights[2]). Sampling at time t: find the bracketing keyframe pair, compute the local parameter, and interpolate. Rotation interpolates with quaternion slerp or nlerp (slerp swings along the rotation's arc at constant speed; nlerp averages the four numbers and rescales them to length 1); translation and scale lerp.

Worked example

A joint's rotation is keyed at 0° at 0.0 s and 90° at 0.5 s, about the same axis. At 0.2 s the local parameter is 0.2 ÷ 0.5 = 0.4. Slerp gives 0.4 × 90° = 36°; nlerp gives 35.5°, half a degree off. If the joint's translation goes from 0 to 1 m over the same keys, it's at 0.4 m.
Don't lerp Euler angles or matrices for rotation

Euler lerp takes a wrong arc and hits gimbal-lock degeneracies; component-wise matrix lerp gives a non-rigid in-between (the same problem as LBS). Quaternion slerp/nlerp is the correct cheap path[8]. Blow frames the slerp-vs-nlerp trade as three properties, of which each method gets two: slerp is constant-velocity and torque-minimal but not commutative; nlerp is commutative and torque-minimal but not constant-velocity. Between two close keyframes the two are nearly identical; for blending several poses (§7), nlerp's commutativity is the property that matters. And check the quaternion dot sign and negate one before interpolating (the double cover), or you go the long way around.
Sample a clip (slerp rotation, lerp translation/scale)
C++ · glm
Rust · glam

Transform sampleTrack(const Track& track, float time) {
    auto [k0, k1, alpha] = track.bracket(time);          // bracketing keys + [0,1] factor
    Transform out;
    out.translation = glm::mix(k0.translation, k1.translation, alpha);   // lerp position
    out.scale       = glm::mix(k0.scale,       k1.scale,       alpha);   // lerp scale
    out.rotation    = glm::slerp(k0.rotation,  k1.rotation,    alpha);   // SLERP rotation
    return out;                                          // glm::slerp handles the shortest path
}

