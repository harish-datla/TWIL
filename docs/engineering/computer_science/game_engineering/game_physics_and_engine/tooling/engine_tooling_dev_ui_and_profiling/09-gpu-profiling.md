# GPU profiling

You wrap the G-buffer pass in a CPU timer and it reports the same tiny number every frame, even when you double the scene. The classic trap: you can't time GPU work with a CPU clock. GPU work is asynchronous: recording and submitting a command buffer returns to the CPU immediately, and the GPU executes later (the GPU Pipeline async model). Wrapping vkCmdDraw in a CPU timer measures recording the command, not executing it.
Use timestamp queries, and read them back later

Ask the GPU to timestamp itself: vkCmdWriteTimestamp writes ticks into a query pool once earlier commands have finished the pipeline stage you name; read back with vkGetQueryPoolResults and convert via timestampPeriod (ns per tick)[19]. The catch: don't read the same frame. VK_QUERY_RESULT_WAIT_BIT blocks the CPU on the GPU and makes your profiler the bottleneck. Ring the pool by frames-in-flight and read a result a few frames old[20]. Also: use VK_QUERY_RESULT_64_BIT (a 32-bit counter at a 1 ns tick overflows in ~4.3 s), mask to timestampValidBits, and reset the pool before reuse. (Vulkan 1.3's vkCmdWriteTimestamp2 does the same job with the newer 64-bit VkPipelineStageFlags2, which add stages[20].)

Worked example

Say timestampPeriod is 40, meaning one tick is 40 ns, and the two timestamps around the G-buffer pass read 1,000,000 and 1,047,500. The pass took 47,500 ticks × 40 ns = 1,900,000 ns = 1.9 ms on the GPU. With three frames in flight at 60 fps, you read that result three frames, about 3 × 16.7 = 50 ms, after you recorded the pass, which is fine for a profiler graph. The 64-bit flag is needed too: a 32-bit counter at 1 ns per tick wraps after 232 ns ≈ 4.3 s.
A GPU timestamp pair around a pass (async readback)
Rust · ash
C++

unsafe {
    device.cmd_reset_query_pool(cmd, pool, base, 2);   // MUST reset before writing
    device.cmd_write_timestamp(cmd, vk::PipelineStageFlags::TOP_OF_PIPE,    pool, base);
    // ...begin rendering, draw the pass (e.g. the deferred G-buffer)...
    device.cmd_write_timestamp(cmd, vk::PipelineStageFlags::BOTTOM_OF_PIPE, pool, base + 1);
}
// A FEW FRAMES LATER (not this frame): read without stalling.
let mut ticks = [0u64; 2];
let read = unsafe { device.get_query_pool_results(pool, old_base, &mut ticks, vk::QueryResultFlags::TYPE_64) };
if read.is_ok() {                                     // Err(NOT_READY): GPU not there yet, try next frame
    let mask = if valid_bits == 64 { u64::MAX } else { (1u64 << valid_bits) - 1 };
    let elapsed_ticks = ticks[1].wrapping_sub(ticks[0]) & mask;   // survives a counter wrap
    let gpu_ms = elapsed_ticks as f64 * timestamp_period as f64 / 1.0e6;
}

Reading a few frames late isn't a nicety. If the CPU reads this frame's timestamps with WAIT_BIT, it sits idle until the GPU finishes the frame, and then the GPU sits idle while the CPU records the next one. The two processors stop working in parallel, so the profiler slows down the frame it's measuring.

Worked example

With 8 ms of CPU work and 11 ms of GPU work per frame, the two overlap and a frame takes the longer of the two, 11 ms (91 fps). Waiting on this frame's timestamps serializes them: 8 + 11 = 19 ms (53 fps), with the GPU idle 8 ms of every frame. With two frames in flight, reading two frames back costs nothing extra. Before recording frame N the CPU already waits on frame N−2's fence so it can reuse that frame's command buffer, and by then N−2's timestamps have been written.
wait this frame
ring of 2
ring of 3
GPU work 11ms
frame 19.0 ms (53 fps)GPU idle 8.0 ms per frameCPU waits 11.0 ms per frame, blocked in the timestamp readtimings shown: this frame, read the moment the GPU finishes it
You're looking at the CPU recording frames (purple) and the GPU running them (orange), replayed in slow motion; a diamond marks the moment the CPU reads a frame's timestamps. Start on wait this frame: after every submit the CPU blocks until the GPU finishes, so the lanes take turns, a frame takes 19 ms, and the GPU idles 8 ms per frame. Switch to ring of 2 or ring of 3: the GPU stays busy, the frame drops to 11 ms, and the timings you display are 2 or 3 frames old. The 3 ms the CPU still waits is the fence of a GPU-bound frame, which happens with or without a profiler. Drag the GPU work below 8 ms and the CPU becomes the slow side.

Per-pass GPU times also don't add up to the frame. A timestamp waits for earlier commands to finish the stage you name, but nothing recorded after it waits for the timestamp, and the implementation may write it at a later stage than the one you asked for[19]. Without a barrier between two passes, the second can start while the first is finishing, so their measured intervals overlap; work on an async compute queue overlaps on purpose. Add up the passes and you count the overlap twice. Time the whole frame with its own pair of timestamps.
Putting GPU zones on the CPU timeline

GPU timestamps count in the GPU's own clock, so you can't draw a GPU zone next to CPU zones until you know how the two clocks line up. VK_KHR_calibrated_timestamps (and the earlier EXT version) returns a GPU and a CPU timestamp sampled together, plus a maximum deviation in nanoseconds, and the specification tells applications to re-calibrate regularly because extrapolating from one sample drifts[21]. Tracy uses the extension for its calibrated Vulkan contexts. OpenGL has no such query, so Tracy's optional periodic re-alignment of an OpenGL context reads the GPU clock with glGetInteger64v, which stalls the pipeline each time it runs[9].
GPU clocks move

GPUs raise and lower their clock speed with load, power and temperature, so the same pass can time differently from one capture to the next. D3D12's SetStablePowerState pins the clocks for profiling. Microsoft's documentation calls it "a development-time aid", says a stable power state "typically fixes GPU clock rates at a slower setting that is significantly lower than that experienced by users", and asks that conclusions drawn under it be double-checked with results from real usage[22]. Use it to compare two versions of a pass; don't quote its absolute times as what players get.
Frame capture is a different tool

Frame debuggers, RenderDoc (free, MIT), PIX (D3D12/Xbox), Nsight (NVIDIA), capture a single frame and let you inspect every draw call, the pipeline state and bound objects at each draw, texture/buffer contents, and shaders, with no code changes[23]. That's offline single-frame dissection ("what exactly did draw #4012 do"), distinct from continuous timestamp timing ("how long is the shadow pass, every frame"). Pipeline-statistics queries (vertex/fragment invocations) sit between them, useful for "why is this pass expensive" (overdraw).
