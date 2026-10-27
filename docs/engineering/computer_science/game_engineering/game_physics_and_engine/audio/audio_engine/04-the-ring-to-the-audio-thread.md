# The Ring to the Audio Thread

The game thread should enqueue a play command; the audio callback dequeues it at the start of a block. A single-producer, single-consumer lock-free ring avoids sharing a mutex across threads. The consumer performs bounded, wait-free work.

```cpp
struct PlayCommand {
    const float* data;
    uint32_t length;
    float gainLeft, gainRight;
};
constexpr size_t kCommandCapacity = 256;
using CommandRing = SpscRing<PlayCommand, kCommandCapacity>;

void requestPlay(CommandRing& ring, const PlayCommand& command) {
    if (!ring.push(command)) { /* drop or count overflow */ }
}

void drainPlayCommands(Mixer* mixer) {
    PlayCommand command;
    for (size_t n = 0; n < kCommandCapacity &&
         mixer->pendingCommands.pop(command); ++n) {
        Voice* voice = findFreeVoice(mixer);
        if (!voice) continue;
        voice->data = command.data; voice->length = command.length;
        voice->gainLeft = command.gainLeft;
        voice->gainRight = command.gainRight;
        voice->position = 0; voice->active = true;
    }
}
```

The lifetime rule is subtle: if the audio thread drops the last `Arc` or `shared_ptr`, its destructor may free memory there. Keep sample ownership on the game thread or send finished buffers back over a second retire ring for the game thread to free. Decide what happens when either ring fills—drop and count, or size for the worst burst. `rtrb` is a suitable Rust SPSC ring for non-`Copy` commands.
