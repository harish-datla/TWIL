# The Platform Layer


You want the same game to run on Windows, Linux and macOS. On each one the operating system owns the window, the input devices, and the stream of events; your engine borrows a drawable region and a feed of events. Each OS exposes them through a different native API: Win32 (Windows' native windowing API), X11 and Wayland (Linux's older and newer display systems), and Cocoa/AppKit (macOS's UI framework)[1].

Engines wrap those differences behind a thin platform layer so the rest of the codebase sees one window type and one event type. The libraries that are that layer for most projects are SDL and GLFW, two cross-platform C libraries, and winit, the standard Rust windowing crate. The framing to hold for the whole tutorial:
A windowing library hands you a window and a native handle, not a renderer

GLFW, SDL, and winit give you a window, an event stream, and an opaque OS handle. They do not draw anything. You bring Vulkan (or OpenGL, or Direct3D) yourself and connect it to that handle. Keeping the window and the renderer decoupled is what lets a Rust engine swap winit for SDL without touching its Vulkan code (§7).

[Back to Platform & Window: the OS Event Loop](index.md)
