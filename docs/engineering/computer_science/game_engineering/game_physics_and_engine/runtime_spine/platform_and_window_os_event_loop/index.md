# Platform & Window: the OS Event Loop

Your game's window says "(Not Responding)" during a level load, freezes while the player drags its border, and looks blurry on a high-DPI laptop. All three come from the platform layer: the code that asks the operating system for a window, pumps its event queue, and hands the GPU a drawable surface. It's small but full of sharp edges, including two completely different control-flow shapes depending on which library you use. This page builds it in C++ and Rust, so you can open a window on any desktop OS, keep it responsive, and connect it to Vulkan.
Time
~80 min
Level
Beginner to mid
Prereqs
The Game Loop tutorial helps (its accumulator gets plugged into the loop here, with a refresher). You can read C++ or Rust.
Stack
C++ (GLFW) · Rust (winit)
◂ Build a Game Engine
Phase 3 · The runtime spine
Next · Input Systems ▸
What you'll be able to do by the end

    Open a window with GLFW or winit and get the native handle a GPU API needs.
    Explain the Win32 message pump, and keep it serviced so Windows never marks your game "(Not Responding)".
    Choose between an owned poll loop and event-driven callbacks, and drop the fixed-timestep accumulator into either.
    Read a monotonic clock and cap the frame rate without the judder a coarse OS timer causes.
    Create a Vulkan surface from the window on Windows, Linux or macOS.
    Handle resizes, high-DPI framebuffers, the modal drag loop and main-thread rules without freezes, blur or crashes, including by hand on Win32.


## Sections

- [The Platform Layer](01-the-platform-layer.md)
- [Creating a Window](02-creating-a-window.md)
- [The OS Event Model](03-the-os-event-model.md)
- [Poll Loop vs Event-Driven](04-poll-loop-vs-event-driven.md)
- [Plugging in the Fixed Timestep](05-plugging-in-the-fixed-timestep.md)
- [Clocks, Sleep & Frame Pacing](06-clocks-sleep-and-frame-pacing.md)
- [Handing a Surface to the GPU](07-handing-a-surface-to-the-gpu.md)
- [Resize, DPI, Threads](08-resize-dpi-threads.md)
- [Pitfalls](09-pitfalls.md)
- [What's Next](10-whats-next.md)
