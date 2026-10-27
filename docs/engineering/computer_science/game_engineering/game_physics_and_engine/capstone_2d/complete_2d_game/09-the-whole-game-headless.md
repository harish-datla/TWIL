# The Whole Game, Headless

The complete headless build can contain the fixed-step loop, scripted input, state machine, uniform-grid broadphase, SPSC ring, real consumer thread, and read-only render-description pass in one file. It leaves out the window and GPU and replaces the clock with a deterministic 144 Hz script containing one 400 ms hitch. That makes output reproducible for tests and diffs.

The key pieces are:

```cpp
template<class T, size_t Capacity>
class SpscRing {
    std::array<T, Capacity> slots{};
    alignas(64) std::atomic<size_t> read{0};
    alignas(64) std::atomic<size_t> write{0};
public:
    bool tryPush(const T& value) {
        size_t w = write.load(std::memory_order_relaxed);
        if (w - read.load(std::memory_order_acquire) == Capacity) return false;
        slots[w % Capacity] = value;
        write.store(w + 1, std::memory_order_release);
        return true;
    }
    bool tryPop(T& value) {
        size_t r = read.load(std::memory_order_relaxed);
        if (r == write.load(std::memory_order_acquire)) return false;
        value = slots[r % Capacity];
        read.store(r + 1, std::memory_order_release);
        return true;
    }
};
```

The scripted run starts the menu, enters playing, spawns waves, auto-aims and fires, resolves grid collisions, builds interpolated quads, and stops the audio consumer before the ring is destroyed. Compile the C++ version with `g++ -std=c++20 -O2 -pthread`; the Rust version can be compiled with `rustc --edition 2021 -O`. The important test properties are deterministic state transitions, bounded simulation catch-up, no simulation-to-GPU calls, dropped rather than blocking audio commands, and consumer shutdown before ring lifetime ends.
