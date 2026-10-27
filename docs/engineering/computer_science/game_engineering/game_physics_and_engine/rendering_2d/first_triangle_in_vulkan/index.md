# Your First Triangle in Vulkan

You have a window and a surface, and you want anything on screen. In Vulkan, a colored triangle takes roughly a thousand lines because you declare the instance, device, queues, swapchain, pipeline, command buffers, and synchronization yourself. This tutorial builds that scaffold in C++ and Rust (ash), using dynamic rendering and two frames in flight.

Time: ~75 min  
Level: Senior  
Prerequisites: The GPU & the Graphics Pipeline and Platform & Window are helpful.  
Stack: C++ (Vulkan) · Rust (ash)

## What you’ll be able to do

- Create and destroy every Vulkan object in validation-approved order.
- Select a GPU, graphics/present queues, swapchain format, extent, image count, and present mode.
- Build a dynamic-rendering pipeline and record its layout barriers.
- Use fences and semaphores correctly with two frames in flight.
- Survive resize and minimize events without hangs or validation errors.

## Sections

- [Why ~1000 Lines](01-why-1000-lines.md)
- [Instance and Validation](02-instance-and-validation.md)
- [Devices and Queues](03-devices-and-queues.md)
- [The Swapchain](04-the-swapchain.md)
- [Shaders to SPIR-V](05-shaders-to-spir-v.md)
- [The Graphics Pipeline](06-the-graphics-pipeline.md)
- [Dynamic Rendering](07-dynamic-rendering.md)
- [Command Buffers](08-command-buffers.md)
- [The Render Loop and Synchronization](09-the-render-loop-and-sync.md)
- [Resize and Shutdown](10-resize-and-shutdown.md)
- [Pitfalls](11-pitfalls.md)
- [What’s Next](12-whats-next.md)
