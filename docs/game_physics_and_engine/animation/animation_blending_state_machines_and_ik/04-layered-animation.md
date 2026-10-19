# Layered animation

A soldier sprints for cover and reloads on the way. Real characters do two things at once: run and reload. Layered animation stacks poses with a bone mask selecting which joints a layer affects. An upper-body layer (aim, reload) plays over a lower-body locomotion layer; the mask is what stops them fighting over the spine and hips.

Besides its pose, a layer has a weight, a mask and a blending type. The mask gives each joint a weight from 0 to 1, which multiplies the layer's weight. Unity's layers take a mask and a blending type, Override or Additive[9]. Unreal's Layered Blend per Bone node picks joints with a branch filter: name a bone, and it and everything below it in the hierarchy get the layer, optionally tapered over a few bones by a blend depth so the spine bends gradually instead of at one joint[6].
Override layers vs additive layers

An override layer replaces the masked joints' pose (up to the layer weight). An additive layer adds a pose difference, a delta from a reference pose (the additive-as-delta rule), composed on top of any base: one aim-offset or lean or recoil delta works over walk, run, and crouch without authoring every combination. A layer is one or the other, and additive is not "playing clip B over clip A"[1].

For rotations the delta is a quaternion. With q⁻¹ meaning the inverse rotation, the open-source ozz-animation library builds an additive clip as delta = reference⁻¹ · pose for every joint and key, and applies it as base · delta, scaling the delta by the layer weight by interpolating it from the identity rotation (no rotation) first[10].

In words: the delta is the extra rotation that turns the reference pose into the additive pose, and applying it turns the base pose by that same extra amount, measured in the joint's own frame. Engines differ on which side the delta multiplies; what matters is using the same convention to build the delta and to apply it.

Worked example

On a 65-joint skeleton, an upper-body mask selects the 24 joints from the spine up. A reload layer at weight 1 replaces those 24 joints' rotations with the reload clip's, and the legs keep running. An additive lean authored as +10° of spine bend from its reference pose then adds 10° to whatever the spine is doing, in the run, the reload, or both. The listing below runs the same idea on six joints: with the upper-body mask, the aim pose sets the arm to 85° while the thigh and shin keep the run's 35° and −60°; with no mask, the legs snap to the aim clip's standing 0° and −5°; and a +12° lean takes the spine from 8° to 20° in the run and from 2° to 14° in the aim.
no mask
spine and up
spine and up, tapered
aim weight 1.00
additive lean
spine 2.0°chest 6.0°arm 95.0°legs: running
Left is the base layer, a run cycle; right is the result with an aim pose layered on top, its bones colored amber where the aim layer drives them. Switch masks, drag the weight, then turn on the lean. With spine and up the arm aims while the legs keep running. With no mask the aim clip, authored standing still, takes the legs too and the runner skates with straight legs. Tapered gives the spine a third of the aim, the chest two thirds and the arm all of it, so the bend spreads over the torso. The additive lean adds 12° to the spine and 3° to the chest on top of whatever the layers produced.
Masked override and additive layers per joint, with quaternions (complete program)
C++
Rust

// Layering poses per joint in local space: an override layer replaces the
// masked joints, an additive layer adds a delta measured from a reference pose.
// Rotations only; translations and scales blend the same way with lerp.
#include <cmath>
#include <cstdio>
#include <numbers>
#include <vector>

struct Quat { float w, x, y, z; };

Quat multiply(Quat a, Quat b) {
    return {a.w * b.w - a.x * b.x - a.y * b.y - a.z * b.z,
            a.w * b.x + a.x * b.w + a.y * b.z - a.z * b.y,
            a.w * b.y - a.x * b.z + a.y * b.w + a.z * b.x,
            a.w * b.z + a.x * b.y - a.y * b.x + a.z * b.w};
}
Quat conjugate(Quat rotation) { return {rotation.w, -rotation.x, -rotation.y, -rotation.z}; }   // the inverse, for a unit quaternion

// Normalized lerp: cheap, and close to slerp for the small angles between poses.
Quat nlerp(Quat a, Quat b, float weight) {
    float dot = a.w * b.w + a.x * b.x + a.y * b.y + a.z * b.z;
    float sign = dot < 0.0f ? -1.0f : 1.0f;              // q and -q are the same rotation: take the short way
    Quat blended{a.w + (sign * b.w - a.w) * weight, a.x + (sign * b.x - a.x) * weight,
                 a.y + (sign * b.y - a.y) * weight, a.z + (sign * b.z - a.z) * weight};
    float length = std::sqrt(blended.w * blended.w + blended.x * blended.x +
                             blended.y * blended.y + blended.z * blended.z);
    return {blended.w / length, blended.x / length, blended.y / length, blended.z / length};
}

// Every joint in this example bends about the x axis, so one angle describes it.
Quat bendX(float degrees) {
    float half = degrees * std::numbers::pi_v<float> / 360.0f;
    return {std::cos(half), std::sin(half), 0.0f, 0.0f};
}
float angleX(Quat rotation) { return 2.0f * std::atan2(rotation.x, rotation.w) * 180.0f / std::numbers::pi_v<float>; }

using Pose = std::vector<Quat>;           // one local rotation per joint
using Mask = std::vector<float>;          // one weight per joint, 0..1

// Override: each joint moves toward the layer's rotation by weight * mask.
Pose overrideLayer(const Pose& base, const Pose& layer, const Mask& mask, float weight) {
    Pose result(base.size());
    for (size_t joint = 0; joint < base.size(); ++joint)
        result[joint] = nlerp(base[joint], layer[joint], weight * mask[joint]);
    return result;
}

// Additive: the delta is what the additive clip changes relative to its own
// reference pose. Scaled by weight * mask, it is applied on top of any base.
Pose additiveLayer(const Pose& base, const Pose& additive, const Pose& reference,
                   const Mask& mask, float weight) {
    const Quat identity{1.0f, 0.0f, 0.0f, 0.0f};
    Pose result(base.size());
    for (size_t joint = 0; joint < base.size(); ++joint) {
        Quat delta = multiply(conjugate(reference[joint]), additive[joint]);
        result[joint] = multiply(base[joint], nlerp(identity, delta, weight * mask[joint]));
    }
    return result;
}

const char* kJointNames[] = {"pelvis", "spine", "chest", "arm", "thigh", "shin"};

void printPose(const char* label, const Pose& pose) {
    std::printf("%-22s", label);
    for (const Quat& rotation : pose) std::printf("%7.1f", angleX(rotation));
    std::printf("\n");
}

int main() {
    // One frame of a run, an aiming pose, and a lean authored against a neutral stance.
    Pose run       = {bendX(5),  bendX(8),  bendX(4),  bendX(-40), bendX(35), bendX(-60)};
    Pose aim       = {bendX(0),  bendX(2),  bendX(6),  bendX(85),  bendX(0),  bendX(-5)};
    Pose leanClip  = {bendX(0),  bendX(12), bendX(3),  bendX(0),   bendX(0),  bendX(0)};
    Pose neutral   = {bendX(0),  bendX(0),  bendX(0),  bendX(0),   bendX(0),  bendX(0)};
    Mask upperBody = {0.0f, 1.0f, 1.0f, 1.0f, 0.0f, 0.0f};   // spine and everything above it
    Mask everything(6, 1.0f);

    std::printf("%-22s", "joint (degrees)");
    for (const char* name : kJointNames) std::printf("%7s", name);
    std::printf("\n");
    printPose("run (base)", run);
    printPose("aim, upper-body mask", overrideLayer(run, aim, upperBody, 1.0f));
    printPose("aim, no mask", overrideLayer(run, aim, everything, 1.0f));
    printPose("aim at weight 0.5", overrideLayer(run, aim, upperBody, 0.5f));
    printPose("run + lean", additiveLayer(run, leanClip, neutral, everything, 1.0f));
    printPose("aim + lean", additiveLayer(overrideLayer(run, aim, upperBody, 1.0f),
                                          leanClip, neutral, everything, 1.0f));
}

