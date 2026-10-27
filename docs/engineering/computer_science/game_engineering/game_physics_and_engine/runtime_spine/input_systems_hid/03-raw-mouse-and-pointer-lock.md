# Raw Mouse & Pointer Lock


In a first-person shooter (FPS), moving the mouse two inches should turn the camera by the same amount whether the player flicks or drags. The cursor doesn't work that way, so there are two mouse signals. The cursor position, for UI, goes through the OS's pointer acceleration (a quick flick moves the cursor farther than a slow drag over the same distance) and the Control Panel mouse-speed slider. Mouselook, steering the camera with the mouse, wants the opposite: raw, unaccelerated deltas, meaning how far the mouse itself moved since its last report.

On Windows, the Raw Input API hands you the device's unfiltered data as WM_INPUT messages, whose mouse deltas, unlike WM_MOUSEMOVE, are "not subject to the effects of the mouse speed set in the Control Panel's Mouse Properties sheet." Check the MOUSE_MOVE_ABSOLUTE flag, though: remote-desktop sessions and pen tablets report absolute positions instead of deltas[5]. winit exposes the raw deltas as DeviceEvent::MouseMotion. For the camera you also lock the cursor so it can't leave the window or stop at the screen edge. winit's CursorGrabMode::Locked pins it in place (you hide it yourself), while Confined only keeps it inside the window. Neither works everywhere (Locked isn't implemented on X11, Confined isn't on macOS), so engines try one and fall back to the other[6].

Worked example

A 1000 Hz mouse reports up to a thousand times a second while it moves: about 17 reports in each 16.7 ms frame at 60 fps (1000 / 60 = 16.7). The player sweeps right steadily at 3 counts per report, where a count is the mouse's own unit of distance. Over one frame that's about 50 counts (16.7 × 3). Sum the deltas and the camera turns by 50 counts' worth. Keep only the latest event and it turns by 3, 6% of what the hand did.
Accumulate deltas, don't overwrite

A 1000 Hz mouse delivers many move events per frame. Store only the latest one and you throw away most of the motion, so sum the deltas across the frame. The point of raw input is deltas without acceleration or speed scaling, which is what a camera wants; it doesn't make input arrive sooner. At high polling rates, GetRawInputBuffer drains the accumulated WM_INPUT events in one batch[5].

[Back to Input Systems (HID)](index.md)
