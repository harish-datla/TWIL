# Input Systems (HID)

A player on a French AZERTY keyboard, where the key in W's spot says Z, can't walk forward. A worn thumbstick, a gamepad's analog stick, makes the character drift while nobody touches it. Slow diagonal aim snaps to eight directions, and a special move fails although the player pressed the right buttons. All four are bugs in the input layer, the code between the operating system and gameplay that handles human interface devices (HID): keyboards, mice and gamepads. This page builds that layer in C++ and Rust as a small pipeline: collect the OS's input events, fold them into one snapshot per frame, clean up the analog values, and translate physical keys and buttons into game actions like Jump.
Time
~55 min
Level
Beginner to mid
Prereqs
Helpful, not required: the Game Loop (input is sampled once per frame) and Platform & Window (the OS events come from there).
Stack
C++ (SDL3) · Rust (gilrs / winit)
◂ Build a Game Engine
Phase 3 · The runtime spine
Next · File Streaming ▸
What you'll be able to do by the end

    Fold the OS's input events into one snapshot per frame, so a tap shorter than a frame still registers.
    Bind movement to physical key positions so WASD works on AZERTY and Dvorak, and feed a camera raw mouse motion.
    Read any gamepad through SDL or gilrs without the DirectInput shared-trigger trap.
    Write a scaled-radial dead zone, a response curve and a frame-rate-independent smoothing filter, and pick numbers for them.
    Put an action map between devices and gameplay, and add an input buffer so timing-sensitive moves come out.


## Sections

- [Where Input Fits](01-where-input-fits.md)
- [Scancodes vs Keycodes](02-scancodes-vs-keycodes.md)
- [Raw Mouse & Pointer Lock](03-raw-mouse-and-pointer-lock.md)
- [Gamepads](04-gamepads.md)
- [Dead Zones](05-dead-zones.md)
- [Response Curves & Smoothing](06-response-curves-and-smoothing.md)
- [Action Maps](07-action-maps.md)
- [Input Buffering](08-input-buffering.md)
- [Sampling & Latency](09-sampling-and-latency.md)
- [Pitfalls](10-pitfalls.md)
- [What's Next](11-whats-next.md)
