# The Render Loop and Synchronization

Semaphores order GPU work; fences synchronize the CPU with the GPU. A binary semaphore is not CPU-waitable. The canonical frame uses an image-available semaphore signaled by acquire, a render-finished semaphore waited on by present, and an in-flight fence signaled when submission completes.

The per-frame order is: wait for the frame fence, acquire an image, reset the fence only once submission is certain, record commands, submit waiting on image-available and signaling render-finished, then present waiting on render-finished. Usually two frames in flight let CPU recording overlap GPU execution. They are not the same as swapchain image count.

```text
frame time ≈ CPU + GPU       (one frame in flight)
frame time ≈ max(CPU, GPU)   (two or more)
```

If recording takes 4 ms and drawing takes 10 ms, one frame in flight takes 14 ms; two can produce a frame every 10 ms. More queued frames add latency once the GPU is already busy.

```rust
device.wait_for_fences(&[in_flight_fence], true, u64::MAX)?;
let image_index = swapchain_loader.acquire_next_image(
    swapchain, u64::MAX, image_available, vk::Fence::null());
// Handle ERROR_OUT_OF_DATE_KHR before resetting the fence.
device.reset_fences(&[in_flight_fence])?;
// Record: barrier → begin rendering → draw → end rendering → barrier.
let submit = vk::SubmitInfo::default()
    .wait_semaphores(&[image_available])
    .wait_dst_stage_mask(&[vk::PipelineStageFlags::COLOR_ATTACHMENT_OUTPUT])
    .command_buffers(&[cmd])
    .signal_semaphores(&[render_finished]);
device.queue_submit(graphics_queue, &[submit], in_flight_fence)?;
let present = vk::PresentInfoKHR::default()
    .wait_semaphores(&[render_finished])
    .swapchains(&[swapchain])
    .image_indices(&[image_index]);
swapchain_loader.queue_present(present_queue, &present)?;
```

FIFO is guaranteed and v-synced. MAILBOX replaces queued images with the newest one for lower latency but may discard rendered frames. IMMEDIATE can tear. Query optional modes and fall back to FIFO.
