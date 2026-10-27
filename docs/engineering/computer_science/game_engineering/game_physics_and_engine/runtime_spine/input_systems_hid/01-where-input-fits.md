# Where Input Fits


A player taps jump for 10 milliseconds. At 60 frames a second a frame lasts 16.7 ms, so if the game only asks "is the button held right now?" at the start of each frame, the whole tap can fall between two checks and the jump never happens. That's why input arrives two ways, and a good engine uses both. The platform layer pushes discrete events (key down, key up, mouse moved, gamepad connected) onto a queue, and the game loop also samples, or polls, each device's current state once per frame: is W held right now? where's the left stick?[1]

The usual shape: at the top of each frame, drain the event queue, fold the transitions into a per-frame snapshot plus an event list, and let gameplay read the snapshot. The events come from the platform layer, the code that owns the window and pumps the OS's message queue. Events are the source of truth for transitions: the exact down and up, even for a tap shorter than a frame. Polling is convenient for "held right now."

Worked example

Frame 1 starts at 0 ms and frame 2 at 16.7 ms. The jump button goes down at 3 ms and up at 13 ms. Polling at 0 ms and at 16.7 ms sees "up" both times, so a polling-only game drops the jump. The event queue holds a key-down at 3 ms and a key-up at 13 ms, so when frame 2 drains it, the snapshot records "pressed this frame" and "released this frame", and gameplay still sees the press.

[Back to Input Systems (HID)](index.md)
