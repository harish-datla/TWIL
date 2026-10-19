# The Real-Time Callback

The audio device pulls samples from the engine on its own clock. A high-priority thread calls a registered callback and gives it a buffer to fill before the hardware needs it. At 48 kHz with a 128-frame buffer, the callback has about 2.67 ms.

At 256 frames, the callback runs 187.5 times per second and has 5.33 ms per call. Mixing 32 stereo voices costs `256 × 32 × 2 = 16,384` multiply-adds. The danger is an occasional stall: an 8 ms lock wait misses a 5.33 ms deadline and produces a click.

## Callback rules

Never call `malloc`, `free`, `new`, `delete`, file or network I/O, or an unbounded loop. An allocator can contend or fault a page; a mutex can block or cause priority inversion. Even `try_lock` is not a general solution because a following unlock may wake a waiting thread through the scheduler. Watch for hidden allocation in `shared_ptr` destruction, `std::function`, and vector growth.

```cpp
void dataCallback(ma_device* device, void* pOutput,
                  const void*, ma_uint32 frameCount) {
    Mixer* mixer = (Mixer*)device->pUserData;
    float* output = (float*)pOutput;
    drainPlayCommands(mixer);              // bounded SPSC pops
    memset(output, 0, sizeof(float) * frameCount * 2);
    for (Voice& voice : mixer->voices) {   // fixed-capacity storage
        if (!voice.active) continue;
        for (ma_uint32 frame = 0; frame < frameCount; ++frame) {
            float sample = voice.data[voice.position];
            output[frame*2]     += sample * voice.gainLeft;
            output[frame*2 + 1] += sample * voice.gainRight;
            if (++voice.position >= voice.length) {
                voice.active = false; break;
            }
        }
    }
}
```

The example omits format negotiation, a master limiter, voice stealing, a retire ring for finished buffers, sample-accurate scheduling, and declick ramps. A lock-free queue is the normal command handoff.
