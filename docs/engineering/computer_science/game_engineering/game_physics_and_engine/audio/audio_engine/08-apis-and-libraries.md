# APIs and Libraries

Every platform has a native low-level path with the same hard deadline but different ownership and wake-up behavior: WASAPI on Windows, CoreAudio/Audio Units on macOS and iOS, and ALSA on Linux. Shared-mode WASAPI uses the OS audio engine; exclusive mode bypasses it. CoreAudio calls a render callback; ALSA writes periods to a PCM ring, blocking or polling.

Cross-platform libraries hide those details. In C++, miniaudio provides a single-file callback-driven layer. In Rust, cpal is a low-level device and stream layer and rodio provides higher-level playback. Shipped games often use FMOD, Wwise, engine audio, or Steam Audio. Under the authoring tools and effects, these systems still contain a real-time voice mixer with the same callback constraints.
