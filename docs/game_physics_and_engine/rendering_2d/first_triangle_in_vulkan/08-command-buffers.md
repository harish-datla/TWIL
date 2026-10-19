# Command Buffers

Vulkan calls are recorded on the CPU into a command buffer and submitted to a queue. Each frame records the image barrier, begin rendering, pipeline bind, dynamic viewport and scissor, `vkCmdDraw(3, 1, 0, 0)`, end rendering, and the present barrier.

A command buffer can only be submitted to a queue from the family its command pool was created for. Command pools are not internally synchronized, so the usual multithreaded pattern is one pool per thread and per frame-in-flight. With two frames and four recording threads, that means eight pools. The triangle’s vertices come from `gl_VertexIndex`, not a bound vertex buffer.
