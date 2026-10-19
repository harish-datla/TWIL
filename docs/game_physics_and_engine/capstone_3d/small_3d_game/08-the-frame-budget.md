# The frame budget

In the model below, the frame fits the budget with 40 enemies and drops to about 38 fps with 100. Which system do you optimize? A model of this engine's frame budget, drawn the way the Tooling module's profiler draws a frame. The slice costs are illustrative, not measured from the arena above. Each subsystem gets a slice; the sim slices repeat per fixed step while the render slices run once. Push the load: the answer to "where do I optimize" is whichever slice is widest.

Worked example

At the widget's defaults (40 enemies, render load 40), AI costs 0.5 + 40 × 0.05 = 2.5 ms, the whole sim group 5.7 ms and the render 9.0 ms: one step and a 14.7 ms frame, inside the budget. At 100 enemies AI is 5.5 ms and the sim group 8.7 ms; one step plus the render would be 17.7 ms, so the accumulator owes a second step and the frame becomes 2 × 8.7 + 9.0 = 26.4 ms, about 38 fps. AI is the widest slice, so that's where to spend. At 300 enemies one sim step alone costs 18.7 ms, more than the 16.7 ms of game time it simulates: the clamp holds it at 15 steps, a 289.5 ms frame (about 3.5 fps), and the game runs at 250 / 289.5 ≈ 86% speed.
Enemies 40
Render load 40
Drag the two sliders and watch the bar against the dashed 16.6 ms line; the readout under the legend gives the frame time and the sim steps per frame. Each colored slice is a subsystem; the sim group repeats once per fixed step the accumulator runs. Crank enemies and the AI slice grows until the frame crosses the 16.6 ms line; that's where to spend (time-slice the pathfinds, fewer per step). Crank render load and lighting/post grow instead. Over budget, the accumulator owes catch-up steps and the sim group repeats: the frame rate drops, but the sim keeps real time as long as one step costs less than 16.6 ms. Past that, every frame owes more steps than it can run, the spiral of death; the loop's 0.25 s clamp caps it at 15 steps a frame, trading a freeze for slow motion.
