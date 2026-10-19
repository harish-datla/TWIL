# Pitfalls

WASD moves wrong on AZERTY
Movement bound to keycodes (symbols) instead of scancodes (positions).
Diagonal aim snaps / dies
A per-axis square dead zone clips diagonals; use scaled-radial.
Stick jumps to a value
Dead zone cut without re-scaling; output never ramps from zero.
Triggers read as one axis
Reading an Xbox pad through DirectInput; use XInput or SDL/gilrs.
Lost mouse motion
Overwriting the per-frame delta instead of summing 1000 Hz events.
Camera feels laggy
A low-pass filter on mouselook; smoothing trades latency for smoothness.
Held key auto-fires
Acting on OS key-repeat events for one-shot actions; check the repeat flag.
Can't rebind keys
Gameplay names physical keys directly instead of going through actions.

[Back to Input Systems (HID)](index.md)
