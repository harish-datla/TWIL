# Why ~1000 Lines

OpenGL hides the instance, device, framebuffer, and synchronization behind global state and the driver. Vulkan is explicit, so a from-scratch colored triangle is on the order of a thousand lines. The payoff is control over CPU-side submission and multithreaded command recording, not a higher frame rate for one triangle.

Libraries provide sensible shortcuts: vk-bootstrap reduces instance, physical-device, device, queue, and swapchain setup; VMA handles memory allocation; wgpu trades low-level control for portability across Vulkan, Metal, D3D12, and OpenGL.

Vulkan lifetimes follow dependencies. Create prerequisites first and destroy dependents first. A typical order is instance → surface → device → swapchain/views → shaders → pipeline → command pool/buffers → semaphores/fences. Shutdown reverses it. Shader modules are an exception: they can be destroyed immediately after pipeline creation because the pipeline does not keep them alive.
