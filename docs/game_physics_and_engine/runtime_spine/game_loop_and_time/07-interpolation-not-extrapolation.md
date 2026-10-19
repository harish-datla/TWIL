# Interpolation, Not Extrapolation


With a 60 Hz simulation on a 144 Hz display, two or three rendered frames in a row can show the same tick, then the picture jumps. Rendered frames fall between simulation states, so the picture stutters. The fix is to keep the previous and current states and blend them with alpha, the leftover accumulator divided by dt[3].

Worked example

A ball moves +1 unit per tick toward a wall at x = 3.5. The previous tick put it at 2.0 and the current tick at 3.0, and this frame's alpha is 0.6.

Interpolation draws 2.0 × 0.4 + 3.0 × 0.6 = 2.6, a position the simulation actually passed through, 0.4 units behind the newest state. Extrapolation predicts forward instead: 3.0 + 1 × 0.6 = 3.6, already 0.1 inside the wall. The next tick bounces the ball back to 3.0, and the extrapolated drawing snaps back 0.6 units.
Render the blended state
C++
Rust

// alpha in [0,1): how far between previous and current we are.
State shown = previousState * (1.0 - alpha) + currentState * alpha;
render(shown);

Because this blends two states that were already computed, it never invents an unphysical position. The cost is latency: the image shows a moment between the last two ticks, so it trails the newest simulated state by up to one fixed step, and real time by more. Unity describes its rigidbody interpolation this way, using "the previous two physics updates" so the body appears "to move slightly behind where it should be"[8]; Godot's docs put the rendered position "between 1 and 2 ticks into the past"[11].
The naming trap: interpolation vs extrapolation

Two of the most-cited game-loop articles blur interpolation and extrapolation. deWiTTERS[5] names its render factor interpolation in code that extrapolates: it predicts forward from the current state and speed (position + speed * interpolation). Nystrom[2] renders the same kind of forward prediction, describes it as "interpolated between where it was on the last frame and where we think it will be on the next frame", and only then calls it extrapolation. Extrapolation adds no latency, but when the prediction is wrong (a collision, a direction change) the next tick corrects it and the object visibly snaps. True interpolation (Fiedler, Unity, Godot) blends past and present and trades latency for never overshooting. When an article says "interpolation", check which of the two its code does.

The widget runs one ball through all three ways of drawing it:
render between sim ticks
Interpolate (past)
Extrapolate (predict)
A ball bounces between two red walls, driven by a fixed simulation that ticks only every 0.28 s so each tick is easy to see. The faint ball labeled "current tick" is the newest simulated state; the purple ball is what gets drawn. Uncheck "render between sim ticks" and the ball jumps once per tick. Interpolate: smooth, but the drawn ball trails the current tick by up to one step. Extrapolate: smooth and on time mid-flight, but at a bounce the drawn ball turns yellow as it shoots through the wall, and snaps back when the next tick corrects it. That snap is why interpolation is the safer default.

[Back to The Game Loop & Time](index.md)
