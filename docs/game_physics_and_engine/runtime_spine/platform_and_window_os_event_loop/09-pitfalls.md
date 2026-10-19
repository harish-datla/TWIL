# Pitfalls

"(Not Responding)"
A long frame or a blocking WndProc starves the pump past 5 seconds; the OS ghosts the window.
Freeze while resizing
Win32 runs a modal pump inside DispatchMessage; render from WM_PAINT/a timer or another thread.
winit: nothing animates
ControlFlow::Wait with no request_redraw(). Use Poll and request a redraw each frame.
winit: deprecated API
The closure EventLoop::run is deprecated in 0.30; use run_app + ApplicationHandler.
Surface creation fails
GLFW window made with a GL context; create it with GLFW_NO_API for Vulkan.
Blurry or cornered on a high-DPI display
A DPI-unaware process gets bitmap-stretched, so it looks blurry; a viewport sized in logical units fills only a corner. Declare Per-Monitor V2 and size the swapchain from the framebuffer in pixels.
Timing with the wall clock
The time of day can jump. Measure frames with a monotonic counter (QPC, steady_clock, Instant).
Sleep-only frame cap
Sleeps end on timer ticks, so frames swing around the target. Sleep short, then spin, or let vsync pace the loop.
Restarting the schedule each frame
Sleeping "one period from now" makes every late frame push the rest back. Advance a fixed deadline instead.
Ignoring WM_DPICHANGED
A Per-Monitor V2 window that doesn't apply the suggested rectangle looks half-size on a 2x monitor.
Crash off the main thread
Creating the window/event loop on a worker thread; keep it on main.

[Back to Platform & Window: the OS Event Loop](index.md)
