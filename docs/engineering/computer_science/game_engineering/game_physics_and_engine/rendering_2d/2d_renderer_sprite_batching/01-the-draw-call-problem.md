# The Draw-Call Problem

Each draw call and state bind has a fixed CPU cost: the driver validates state, translates it, and encodes commands. One draw per sprite can make submission rate, not rasterization, set the frame rate.

At an illustrative 25,000 batches per second, the CPU can submit only about 416 batches per frame at 60 fps. Two thousand sprites issued individually would require about 80 ms of submission time on that old measurement—roughly 12 fps—regardless of GPU speed. One batch reduces that portion to one call.

Batching primarily speeds the CPU and driver. It does not magically make the GPU shade fewer pixels, though it can reduce state changes and total work. With three textures in shuffled order, a batch may break hundreds of times; sorting by texture can reduce it to three draws when order permits. An atlas makes many sprites share one texture and therefore one draw without changing their visual order.
