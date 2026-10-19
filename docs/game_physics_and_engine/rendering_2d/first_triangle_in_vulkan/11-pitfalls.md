# Pitfalls

- **Waiting on a semaphore from the CPU:** Binary semaphores order GPU work. The CPU waits on a fence or timeline semaphore.
- **Assuming MAILBOX exists:** Only FIFO is guaranteed; query present modes.
- **Blank screen with dynamic rendering:** Add the explicit `UNDEFINED → COLOR_ATTACHMENT_OPTIMAL → PRESENT_SRC_KHR` barriers.
- **Putting the swapchain extension in instance extensions:** Surface extensions are instance-level; swapchain is device-level.
- **Passing the same queue family twice:** Deduplicate graphics and present indices.
- **Recreating the pipeline on every resize:** Make viewport and scissor dynamic.
- **Equating frames in flight with image count:** Frames in flight duplicate command buffers and sync objects; the swapchain has its own image count.
- **Validation errors at exit:** Wait for the device and destroy in reverse order.
- **Hanging after resize:** Reset the in-flight fence only once an image acquisition will lead to a submission. Resetting before an out-of-date early return leaves it unsignaled forever.
