# Poll Loop vs Event-Driven


A game wants to redraw 60 or more times a second whether or not the player touches anything. A level editor sitting idle shouldn't burn a CPU core redrawing the same picture. A platform layer can take two shapes, and they decide where your loop lives and whether it can sleep.

    Owned poll loop (SDL, GLFW): you own while (running). Each frame you call poll-events to drain the queue, then update and render. The library is a service you call[6][7].
    Event-driven callbacks (winit; also SDL3's main-callback mode): the framework owns the loop and calls you. You implement handlers and hand control back every iteration[8][9]. winit 0.30 moved to this trait-based API, deprecating the older closure passed to EventLoop::run[10].

The callback shape exists because some platforms (iOS, the web) require the system to own the loop and call into you, and on others (macOS live resize, Win32 modal loops) the system takes the loop over at times anyway. The poll loop is the convenient shape where the platform allows it. GLFW frames its choice plainly: glfwPollEvents "processes only those events that have already been received and then returns immediately... the best choice when rendering continuously, like most games do," while glfwWaitEvents sleeps until something happens, which suits tools and editors[6]. winit makes the same choice through ControlFlow: Wait, the default, sleeps until an event arrives, and Poll keeps the loop running for games[11].

The widget feeds the same event stream into both shapes:

Worked example

A game under Poll with vsync (presenting each image in step with the display's refresh) on a 60 Hz display draws 60 frames a second, input or not. An editor under Wait whose user presses a key every 1.6 seconds draws one frame per key press, about 0.6 frames a second, and its loop sleeps the rest of the time. The same editor under Poll would draw 60, almost all of them identical.
Event-driven: Poll
Event-driven: Wait
Left is a poll loop you own, stepping through poll, update and render; right is a framework that owns the loop and calls your two handlers. The counters under each column count frames drawn. Under the default Poll both climb about 60 a second. Press "Event-driven: Wait" and the right column reads idle and its counter stops, then goes up by one each time an event arrives (one every 1.6 s in this demo) while the left keeps climbing. That's why a game sets Poll and asks for a redraw each frame.

[Back to Platform & Window: the OS Event Loop](index.md)
