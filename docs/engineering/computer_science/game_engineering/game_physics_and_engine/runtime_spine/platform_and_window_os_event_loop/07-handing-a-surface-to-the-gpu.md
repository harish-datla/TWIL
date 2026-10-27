# Handing a Surface to the GPU


Your Vulkan renderer runs on Windows and you want it on a Wayland desktop too, without rewriting the drawing code. That works because Vulkan's core knows nothing about windows; presentation comes through the WSI (Window System Integration) extensions. The cross-platform VK_KHR_surface defines the VkSurfaceKHR type, and one platform extension (VK_KHR_win32_surface, VK_KHR_xcb_surface, VK_KHR_wayland_surface, a Metal surface on macOS) actually creates it from the native handle[20].
Rendering is portable; surface creation is platform-specific by design

Both halves are true at once. Once the surface (the Vulkan object that stands for the window's drawable area) exists, the swapchain (the set of images that take turns on screen) and all the drawing are platform-independent, but the call that makes the surface is tied to the OS. The libraries hide it: GLFW's glfwCreateWindowSurface picks the right extension for you (and requires the window be created with GLFW_NO_API, proof again that GLFW gives you a surface, not a renderer)[21]. In Rust, raw-window-handle, a tiny crate that defines one handle type both sides understand, decouples the window from the GPU library, and ash-window creates the surface from that handle[22][23].

Worked example

On Windows the Vulkan instance needs two extensions enabled before a surface can exist: VK_KHR_surface, which defines the VkSurfaceKHR type, and VK_KHR_win32_surface, which creates one from an HWND. On a Wayland desktop the second becomes VK_KHR_wayland_surface, and on X11 VK_KHR_xcb_surface[20]. Those platform pieces, one extension name and the one call that creates the surface (GLFW or ash-window picks both for you), are all that change; the swapchain, pipelines and draw calls built on the surface are the same on all three.
Create the surface (overview; the swapchain is the Vulkan module)
C++
Rust

VkSurfaceKHR surface;
// GLFW selects VK_KHR_win32_surface / _xcb_ / _wayland_ for you.
if (glfwCreateWindowSurface(instance, window, nullptr, &surface) != VK_SUCCESS)
    throw std::runtime_error("failed to create surface");


[Back to Platform & Window: the OS Event Loop](index.md)
