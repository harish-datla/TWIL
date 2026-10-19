# The Shape of `main()`

Initialize in dependency order and shut down in reverse. A typical order is platform → input → renderer → assets → audio → world → state machine. The audio thread must stop before its ring is freed; GPU objects must be destroyed before the device.

```cpp
int main() {
    Platform platform;
    InputState input;
    Renderer renderer(platform);
    Assets assets("assets.pak");
    SpscRing<AudioCommand, 1024> audioRing;
    AudioEngine audio(audioRing);
    World world = loadArena(assets, renderer);
    GameState state = GameState::Menu;

    const double dt = 1.0 / 60.0;
    double accumulator = 0.0;
    auto currentTime = Clock::now();
    while (platform.running()) {
        auto now = Clock::now();
        double frameTime = duration<double>(now - currentTime).count();
        frameTime = std::min(frameTime, 0.25); // spiral guard
        currentTime = now;
        platform.pumpEvents(input);
        accumulator += frameTime;
        while (accumulator >= dt) {
            world.savePreviousTransforms();
            simulate(world, input, audioRing, dt, state);
            accumulator -= dt;
        }
        buildBatch(world, float(accumulator / dt), renderer.batch());
        renderer.render();
    }
}
```

Scope destruction reverses declaration order: world, audio, ring, assets, renderer, platform. Production additions such as resize recovery, asset eviction, CCD, hot-plug, voice stealing, and save/load belong in their respective modules.
