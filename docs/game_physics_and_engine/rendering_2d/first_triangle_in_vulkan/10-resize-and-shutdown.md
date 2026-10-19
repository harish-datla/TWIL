# Resize and Shutdown

`VK_ERROR_OUT_OF_DATE_KHR` means the swapchain must be recreated. `VK_SUBOPTIMAL_KHR` is a successful result that should trigger recreation soon. Do not treat every result other than `VK_SUCCESS` as an error; the two codes have opposite signs.

Keep a framebuffer-resized flag from the window callback because some platforms do not report the error codes reliably. A minimized window can have extent `(0, 0)`, so pause and wait for events until the extent is nonzero. Before recreation, call `vkDeviceWaitIdle`, destroy old swapchain-dependent objects in reverse order, and rebuild them.

Shutdown also waits for the device and destroys objects in reverse dependency order. Rust/ash and the C API require explicit destruction; Vulkan-Hpp RAII wrappers destroy handles automatically according to declaration order.
