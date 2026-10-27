# Creating a Window


glfwCreateWindow(1280, 720, "Engine", nullptr, nullptr) gives you a window on screen, and nothing draws in it until a renderer is attached. Underneath that one call, every platform does the same three things: describe the window, create it, and get back an opaque handle, a token you pass back to every later call that concerns that window (HWND on Windows, an X11 window id, a Wayland surface, an NSWindow). A library normalizes that into one call.

The handle is the thing you hand to the GPU later for surface creation, so creating the window and creating the renderer are two separate steps even though they feel like one. §5 has complete GLFW and winit programs that open a window and run the game loop, and §8 shows the same thing written directly against Win32, which is what those libraries do for you on Windows.

[Back to Platform & Window: the OS Event Loop](index.md)
