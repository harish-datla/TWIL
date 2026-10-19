# Sampling & Latency


A press that lands just after the frame's input check waits a whole frame before the game sees it. Polling once per frame can also miss a transition shorter than the frame entirely, like the 10 ms tap in §1. The event stream carries every transition with a timestamp, which is why fast-twitch genres lean on events (and on sub-frame sampling, like SF6's three reads per frame) rather than per-frame polling alone.

Worked example

Presses arrive at random moments, so on average a press waits half a frame for the next sample. At 60 fps that's 8.3 ms on average and 16.7 ms at worst. At 144 Hz it's 3.5 ms and 6.9 ms.

That wait is one stage of input latency, the delay from a physical press to the result on screen. The render queue and the display add their own delays, so sampling adds up to about half a frame on average (a full frame worst case) on top of the rest. High-frequency mice deliver many events per frame; accumulate them rather than taking the last (§3). Timestamps let you order inputs and age the buffer from §8, which is a timestamped ring of recent inputs.

[Back to Input Systems (HID)](index.md)
