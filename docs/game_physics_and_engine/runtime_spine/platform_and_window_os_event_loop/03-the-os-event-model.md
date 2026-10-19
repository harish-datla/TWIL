# The OS Event Model


A level load blocks the main thread for six seconds, and a second before it finishes Windows swaps the window for a faded ghost copy with "(Not Responding)" in the title bar. That happens because the OS doesn't call your code when a key goes down. It puts a message in a queue and waits for you to come get it. On Windows the loop that collects them is the message pump: retrieve a message, translate it, dispatch it to your window procedure, the callback you registered to handle that window's messages[2].

The pipeline below is a sequence of Win32 calls, not arithmetic, and every thread that owns a window runs some version of it:
eq. 1 · the Win32 message loop GetMessage / PeekMessage → TranslateMessage → DispatchMessage → WndProc

In words: pull a message off the queue (GetMessage waits for one, PeekMessage returns at once), optionally turn key presses into characters, then dispatch it so the OS calls your WndProc. Each arrow means "then". Hover any name to see what it does.

    GetMessage blocks until a message arrives and returns 0 on WM_QUIT (and −1 on error, which is why the idiom is while (GetMessage(...) > 0))[3]. Games typically use PeekMessage with PM_REMOVE instead, which is non-blocking, so the loop can render every iteration whether or not events are waiting[4].
    TranslateMessage only turns keyboard virtual-key messages (layout-independent key codes such as VK_LEFT) into character messages; it does not "translate all messages."
    DispatchMessage calls your WndProc for that message and returns when it returns[2].

Linux and macOS differ in detail but not in spirit. X11 is a connection you pull events off; Wayland is a protocol where the compositor, the process that draws every window onto the screen, requests a size and the client must acknowledge it. macOS's NSApplication owns its run loop and dispatches NSEvents; a library can pump that loop by hand (GLFW and SDL do), but during a live resize AppKit runs its own tracking loop, much like the Win32 modal loop. Platforms like these, plus iOS and the web, are why the cross-platform Rust library (winit) owns your loop outright (§4).
Service the pump or the OS declares you hung

Windows attaches a timer to pending messages: if one sits in a window's queue for 5 seconds without being retrieved, the Desktop Window Manager (DWM, Windows' compositor) declares the window hung and replaces it with a ghost copy showing a bitmap of its last contents and "(Not Responding)" in the title[5]. The OS can't tell "busy" from "hung"; it only knows messages aren't being retrieved. A single long frame, or a blocking call inside WndProc, is enough.

Worked example

The six-second load runs on the main thread, with no pump. The player keeps clicking and pressing keys, about 8 input events a second. At 0.5 s, 4 events are waiting; at 4.5 s, 36. At 5.0 s there are 40, and the oldest has sat unretrieved for 5 seconds, so Windows ghosts the window. At 6.0 s the load ends, the next pump drains all 48 at once, and the window comes back. Pumping once inside the load loop, even every half second, keeps the oldest message under 5 s old and the window never ghosts.

The widget is the pump, running that example:
Event rate 8/s
Frame cost 40 ms
The blue dots are the OS event queue, the box below it is the pump, and the readouts show the queue length and the time since the last pump. At the default 40 ms frame the pump runs every frame and the readouts sit at "queued events: 0" and "0.0 s". Drag the frame cost to 6000 ms (one long, blocking frame after another) and events pile up between pumps: at 8 events a second the readout passes 4, 8, 12 every half second and reaches 40 at 5.0 s; once five seconds pass without a message being retrieved, the title flips to "(Not Responding)" and the window turns red (Windows shows a faded ghost copy instead), the Win32 ghosting rule. At 6 s the pump drains the queue and it starts over. The event rate only changes how many events pile up; the hang rule is about how long the oldest one has waited. Lower the cost and it recovers.

[Back to Platform & Window: the OS Event Loop](index.md)
