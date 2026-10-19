# Timing Sources


A player changes the system clock while the game runs, and physics gets a negative dt; a frame limiter calls Sleep to wait out the rest of a frame and misses the next refresh. Both come from where the loop gets its time. Measure frame time with a monotonic clock, one that never goes backward, never a wall clock.

    Use std::chrono::steady_clock in C++ and std::time::Instant in Rust. Both are monotonic and meant for measuring intervals[13][14]. A wall clock (system_clock) can jump backward when NTP steps it or a user changes the time, and hand you a negative dt. (Daylight saving doesn't move it: system_clock counts UTC, and DST only changes how local time is displayed.)
    sleep() is not precise. On Windows the default timer granularity is about 15.6 ms, so Sleep(1) can sleep roughly 15 ms[15]. A common approach to precise frame pacing (delivering frames at even intervals) sleeps to near the target, then spin-waits the rest, checking the clock in a tight loop.
    With vsync on, the present call blocks once the swap chain is full, until a refresh frees an image, so presentation itself paces the loop. The accumulator absorbs that cadence, and absorbs the jitter, the frame-to-frame unevenness, when a frame is dropped or repeated under load.

Worked example

A 60 fps frame has 16.7 ms. The game's work takes 10 ms, so the limiter calls Sleep(6). With Windows' default 15.6 ms timer granularity, that sleep can last up to about 15.6 ms, and the frame takes up to 10 + 15.6 = 25.6 ms, about 39 fps. Spin-waiting the last 6.7 ms instead hits 16.7 ms at the cost of a busy core; raising the timer resolution with timeBeginPeriod lets a short sleep cover most of that wait first[15].

Separately, if NTP steps the wall clock back 2 s during a 16.7 ms frame, a system_clock difference reports dt = 0.0167 − 2 = −1.983 s. The listing's frameTime > 0.25 clamp doesn't catch a negative value. steady_clock and Instant can't go backward, so it can't happen.
A note on the windowing layer

In an SDL or GLFW program you own the while loop and the accumulator drops straight in. In Rust's winit the framework owns the loop and calls you back: keep the accumulator in your handler, set ControlFlow::Poll, do the frame work in the RedrawRequested event rather than in about_to_wait, and call request_redraw() every frame so the next RedrawRequested arrives[16]. Same algorithm, inverted control flow. The Platform & Window tutorial builds this out.

[Back to The Game Loop & Time](index.md)
