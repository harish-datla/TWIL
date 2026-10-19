# The state machine

The player pushes the stick from rest to full, and the character has to go from idle to walk to run without a jolt at each change. An animation state machine is the standard hand-authored model (state machines inside Unreal's AnimGraph, Unity's Animator Controllers)[1][2][3]. Each state produces one pose (a clip or a blend tree). Each transition is {toState, condition, blendDuration}. A transition is a timed cross-fade between the source state's pose and the destination's, with weight clamp(elapsed / blendDuration, 0, 1) in the simplest (linear) case: the local-space cross-fade from Skeletal Animation, driven by time instead of a slider.

Worked example

A walk-to-run transition with a 0.2 s blend at 60 fps lasts 12 frames, with the weight climbing by 1 ÷ 12 = 0.083 each frame. If the knee sits at 20° in the walk and 45° in the run at that moment, the cross-fade moves it about 2° a frame. A hard cut moves it the whole 25° between two frames, which reads as a pop. For those 12 frames the cross-fade samples both the walk and the run; inertialization samples only the run.
A transition is not an instant switch, and inertialization is the cheaper alternative

A hard cut (weight jumps 0→1) pops: joint angles jump between two frames. Blend in local space per joint, then recompute globals; blending global matrices gives non-rigid, shrinking in-betweens (the LBS failure). A dual-pose cross-fade evaluates both states during the transition (double cost). A production alternative, offered by Unreal as a transition blend mode[2], is inertialization (Bollo, Gears of War 4): switch to the destination immediately, record the offset from the source pose, and decay that offset to zero over the transition, so only one pose graph is evaluated[4]. Blend durations are authored per transition; fast reactions use shorter blends than locomotion changes.

The widget runs a three-state machine driven by one speed input. Its blend lasts 0.4 s, slower than most games use, so the fade is easy to watch:
Speed
hard cut (no blend)
The three states (idle, walk, run) with the active one highlighted, a blend-weight bar, and a stick figure playing the blended gait. Drag Speed up and down, then tick hard cut and do it again. Speed above 20 selects walk and above 65 selects run. Raise speed to cross thresholds (idle → walk → run); the transition plays as a cross-fade (the weight bar fills over the blend duration) and the figure's gait sweeps smoothly. Enable hard cut and the figure pops at each change, the instant-switch artifact the blend exists to hide.
An animation state machine (transitions are timed cross-fades)
C++
Rust

struct Transition { int toState; ConditionFn condition; float blendDuration; };
struct State { PoseSource source; std::vector<Transition> transitions; };  // source = clip or blend tree

Pose StateMachine::evaluate(float dt, const Params& params) {
    if (targetState < 0)                                  // idle: check transitions in priority order
        for (auto& transition : states[currentState].transitions)
            if (transition.condition(params)) {
                targetState = transition.toState; elapsed = 0;
                activeBlendDuration = transition.blendDuration; break;
            }

    Pose from = states[currentState].source.sample(dt, params);  // local-space pose
    if (targetState < 0) return from;

    Pose to = states[targetState].source.sample(dt, params);  // both states run mid-transition
    elapsed += dt;
    float weight = clamp(elapsed / activeBlendDuration, 0.0f, 1.0f);
    Pose blended = crossFade(from, to, weight);          // per-joint slerp/lerp, LOCAL space
    if (weight >= 1.0f) { currentState = targetState; targetState = -1; }  // done
    return blended;
}

Inertialization: one pose instead of two

A cross-fade samples both clips for the whole blend, and in a real graph each "clip" can be a blend tree of its own, so a busy transition roughly doubles the animation cost. Inertialization switches to the new clip at once and hides the jump with an offset: on the switch frame, record how far the new clip's pose is from the pose on screen, add that offset to the new clip, and shrink it to zero over the next few frames[4]. After the switch frame, only the destination is sampled. Bollo's original presentation shrinks the offset along a polynomial over a fixed time; Holden's version, used below, decays it with a critically damped spring instead, which has no fixed end time and makes the code a few lines[5]. Unreal offers inertialization as a blend mode on state-machine transitions[2].

Worked example

From the listing's output: at t = 0.50 s the walk clip has the knee at 20.0° and the run clip at 37.4°. A hard cut shows 37.4° on that frame, a 17.4° jump. The inertializer shows 20.0° (run plus an offset of −17.4°), then 30.5° at 0.55 s and 50.2° at 0.60 s as the offset decays with an 80 ms halflife, and it's within 1.2° of the run clip by 0.80 s. The 0.25 s cross-fade also starts at 20.0°, but it samples both clips until 0.75 s.
hard cut
cross-fade
inertialize
switch clip
auto-switch
playing: runclips sampled this frame: 1break in the curve at the switch: 0.1°settled within 1° of the new clip after: 300 ms
One knee angle over time: the two dashed curves are the walk and run clips, the solid line is what the character shows, and a purple line marks each switch (every 1.5 s, or press switch clip). Compare the three modes and watch the readout. A hard cut jumps straight onto the other clip, and the break in the curve reads the full gap between them. A cross-fade starts where the old clip was and bends onto the new one over 0.25 s, sampling 2 clips while it does. Inertialize keeps the curve continuous, settles within 1° of the new clip in under 0.4 s, and after the switch frame samples only 1 clip. When the clips cross at the switch but move at different speeds, the offset starts near zero and still swings out a few degrees and back: that swing is the old clip's velocity being carried through. Press switch twice in quick succession: the inertializer adds the new offset to the one still decaying, so it stays smooth.
Inertialization for one channel, against a hard cut and a cross-fade (complete program)
C++
Rust

// Inertialization for one animation channel (a knee angle), compared with a
// hard cut and a cross-fade. The offset decays on a critically damped spring.
#include <cmath>
#include <cstdio>
#include <numbers>

constexpr float kPi = std::numbers::pi_v<float>;

// Halflife sets how fast the offset dies away (smaller is faster). It is named after a
// simple damper's time to halve the gap; a spring carrying velocity is only roughly there.
float halflifeToDamping(float halflife) { return (4.0f * std::numbers::ln2_v<float>) / (halflife + 1e-5f); }

// Move an offset and its velocity toward zero with a critically damped spring,
// solved exactly for the step dt, so it is stable at any frame rate.
void decaySpring(float& offset, float& offsetVelocity, float halflife, float dt) {
    // Closed form: offset(t) = e^(-decayRate t) * (offset + linearTerm * t).
    float decayRate = halflifeToDamping(halflife) / 2.0f;
    float linearTerm = offsetVelocity + offset * decayRate;
    float decay = std::exp(-decayRate * dt);
    offset = decay * (offset + linearTerm * dt);
    offsetVelocity = decay * (offsetVelocity - linearTerm * decayRate * dt);
}

struct Inertializer {
    float offset = 0.0f;
    float offsetVelocity = 0.0f;

    // At the switch, record how far the new clip is from what is on screen.
    // Adding the old offset lets a switch interrupt a switch still in progress.
    void transition(float sourceValue, float sourceVelocity, float destValue, float destVelocity) {
        offset = (sourceValue + offset) - destValue;
        offsetVelocity = (sourceVelocity + offsetVelocity) - destVelocity;
    }

    // Each frame: show the destination plus the offset, then shrink the offset.
    // Only the destination clip is sampled.
    float update(float destValue, float halflife, float dt) {
        float shown = destValue + offset;
        decaySpring(offset, offsetVelocity, halflife, dt);
        return shown;
    }
};

// Two clips' knee angle in degrees, and its rate of change in degrees per second.
float walkKnee(float time) { return 20.0f + 15.0f * std::sin(2.0f * kPi * 1.0f * time); }
float walkKneeVelocity(float time) { return 15.0f * 2.0f * kPi * 1.0f * std::cos(2.0f * kPi * 1.0f * time); }
float runKnee(float time) { return 45.0f + 30.0f * std::sin(2.0f * kPi * 1.6f * time + 1.0f); }
float runKneeVelocity(float time) { return 30.0f * 2.0f * kPi * 1.6f * std::cos(2.0f * kPi * 1.6f * time + 1.0f); }

int main() {
    const float dt = 1.0f / 60.0f;
    const int switchFrame = 30;                 // walk until t = 0.5 s, then run
    const float crossFadeTime = 0.25f, halflife = 0.08f;
    Inertializer inertializer;

    std::printf("    t   walk    run   cut  cross  inert\n");
    for (int frame = 24; frame <= 60; ++frame) {
        float time = frame * dt;
        float walk = walkKnee(time), run = runKnee(time);
        bool running = frame >= switchFrame;
        if (frame == switchFrame)
            inertializer.transition(walk, walkKneeVelocity(time), run, runKneeVelocity(time));

        float hardCut = running ? run : walk;
        // Cross-fade: both clips are sampled for the whole blend.
        float weight = running ? std::fmin(1.0f, (time - switchFrame * dt) / crossFadeTime) : 0.0f;
        float crossFade = walk + (run - walk) * weight;
        float inertial = running ? inertializer.update(run, halflife, dt) : walk;

        if (frame % 3 == 0)
            std::printf("%5.2f %6.1f %6.1f %5.1f %6.1f %6.1f\n", time, walk, run, hardCut, crossFade, inertial);
    }
}

A real implementation runs this per joint: the rotation offset is a quaternion (the difference between the two rotations) decayed toward the identity, the rotation that does nothing, and the translation offset is a vector, each with its own velocity. Velocity matters at the switch: the offset starts with the difference in velocity as well as position, so the knee keeps the speed it had on the old clip through the switch instead of snapping to the new clip's speed. Unreal also has an experimental dead blending node[6], a variant that extrapolates the old pose forward from its velocity and blends from that into the new clip, so at the switch it needs only the current pose and its velocity[7].
