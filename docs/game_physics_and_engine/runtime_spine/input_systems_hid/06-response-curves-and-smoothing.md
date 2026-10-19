# Response Curves & Smoothing


A sniper scope in a console shooter needs tiny, precise turns from small stick movements, and a worn stick or a gyro sensor adds jitter on top. Two more shaping steps handle this, and both run on the stick's length after the dead zone. A response curve bends sensitivity: output = lengthk with k > 1 shrinks small pushes while full deflection stays at 1, so aim gets finer near the center. A low-pass filter smooths jitter by letting slow changes through and damping fast ones.

Worked example: the curve

A half-pushed stick has length 0.5. After the 0.25 dead zone that's (0.5 − 0.25) / 0.75 = 0.33. With k = 2 the curve squares it to 0.11. Full deflection still gives 1 (12 = 1), so the player keeps top speed, and a half push now moves at a third of the speed it did without the curve.

The filter is a single line, an exponentially weighted moving average: a running average where each step back in time multiplies a reading's weight by the same factor, 1 − α, so old readings fade out[16]. It computes each frame's output as a blend of the new reading and last frame's output, so a sudden jump only moves the result part of the way:
eq. 2 · one-pole low-pass yi = α · xi + (1 − α) · yi−1, 0 ≤ α ≤ 1

In words: each frame, move the smoothed value a fraction α of the way from where it was toward the new reading. At α = 1 there's no smoothing (output equals input); push α toward 0 and the output leans on its own history, so it's smoother but trails the input. The subscript i−1 means "the value from the previous frame".

Worked example: the filter

With α = 0.2, the player slams the stick from 0 to 1. The filtered value goes 0.20, 0.36, 0.49 over the first three frames and passes 90% on frame 11, about 183 ms at 60 frames a second. Run the same α = 0.2 at 144 Hz and it still passes 90% on frame 11, but that's now 76 ms: the smoothing got weaker because the frames got shorter. The compensated weight at 144 Hz is 1 − 0.860/144 = 0.089, which takes 25 frames, about 174 ms, close to the 60 Hz feel.
Smoothing trades latency for smoothness

A smaller α smooths harder but makes the filtered signal lag the input more. Use it for drifty sticks or gyro; avoid it on anything where input lag is felt directly, like a mouselook camera. If your frame time varies, compute α each frame from a smoothing time constant τ, α = 1 − e−Δt/τ, or the smoothing strength drifts with the frame rate. A τ of about 75 ms gives α = 0.2 at 60 Hz and about 0.09 at 144 Hz, the numbers in the example. Unreal's Enhanced Input ships both steps as per-binding modifiers (an exponential response curve and a smoothing modifier)[12]. Unity's Input System Processors cover dead zones, scaling and inversion, and anything more is a custom processor[13].

The widget feeds a noisy stick signal through both steps:
Response curve k 1.0
Smoothing α 1.00
Top: the response curve, with raw length across and output up; the dotted diagonal is k = 1. Drag k to 2.0 and the curve sags below the diagonal: small pushes give smaller outputs, so aim is finer near the center, while full deflection still reaches the top corner. Bottom: a noisy raw signal (blue) and its filtered output (violet), scrolling. At α = 1.00 the two lines coincide. Drop α to 0.05, the minimum, and the violet line loses the fast wiggle and runs behind every swing. That trailing is the latency you trade for smoothness. α is the weight per 60 Hz step, compensated for the display's real frame time, so the smoothing looks the same on a 144 Hz screen.

[Back to Input Systems (HID)](index.md)
