# Scancodes vs Keycodes


Your game binds forward to W. A player in France has an AZERTY keyboard, where the key in W's spot is labeled Z and the W key sits bottom-left. Whether forward still works under their left hand depends on which of two codes you bound. A scancode identifies the physical key position, whatever the layout. A keycode, also called a virtual key, names the symbol the key produces under the current layout. SDL's keyboard guide is explicit: scancodes "are guaranteed to reference the physical location on the keyboard and not what is printed on it," while keycodes "name specific keys and don't care where on the user's keyboard they actually are"[3]. SDL's scancode values come straight from the USB HID usage tables, the standard numbering of key positions that USB keyboards report[2].

Worked example

The AZERTY player presses the key under their left middle finger, the one QWERTY labels W and AZERTY labels Z. SDL reports scancode SDL_SCANCODE_W (26, the HID usage ID for that position) and keycode SDLK_Z. Bound to the scancode, forward works. Bound to the keycode for W, nothing happens: the player has to reach for the bottom-left key, where QWERTY has Z, and strafe left with the key in QWERTY's Q spot.
Bind movement to scancodes, shortcuts to keycodes

Bind WASD to keycodes and the AZERTY player's keys in the WASD spots, labeled Z, Q, S and D, produce the wrong symbols for forward and strafe-left. Bind movement to scancodes and the keys under the left hand work on any layout. Symbolic shortcuts ("press I for inventory") want keycodes, so the letter matches the on-screen prompt[3]. winit draws the same line: physical_key (a KeyCode, the position) vs logical_key (a Key, the meaning), and its logical key is deliberately not affected by Ctrl, so Ctrl+C still reports the character "c"[4].

One more keyboard detail. Hold a key in a text box and, after a short delay, the letter repeats. That's auto-repeat: the OS keeps resending key-down events while the key stays held. It's a typing convenience, and gameplay ignores it. One-shot actions fire only when the event isn't a repeat (SDL's repeat flag, winit's KeyEvent.repeat), and held movement comes from polling state each frame[4].

[Back to Input Systems (HID)](index.md)
