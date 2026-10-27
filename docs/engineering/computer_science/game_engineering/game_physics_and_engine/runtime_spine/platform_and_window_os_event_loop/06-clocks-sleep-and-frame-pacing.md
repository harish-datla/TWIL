# Clocks, Sleep & Frame Pacing


Your game sits on its menu screen with vsync off, and one CPU core runs at 100% drawing the same picture a thousand times a second while the laptop fan spins up. The obvious fix, sleeping until the next frame is due, can turn a 60 fps cap on Windows into frames that swing between a few milliseconds and 31. Two platform facts decide this: which clock you read, and how precisely the OS can wake a sleeping thread.

The clock. Frame times come from a monotonic high-resolution counter, never the time of day, which can jump when the system clock is corrected. On Windows that's QueryPerformanceCounter (QPC), which Microsoft recommends for time stamps of 1 microsecond resolution or better that don't need to match an outside time reference[12]. C++'s std::chrono::steady_clock and Rust's Instant are portable wrappers over the same kind of counter, and GLFW's glfwGetTime in §5 reads one too.

The sleep. A sleeping thread wakes on a timer interrupt, which rarely falls at the exact instant it asked for. Microsoft's documentation for Sleep says the system clock "ticks" at a constant rate, and a wait of more than one tick but less than two can last anywhere between one and two ticks[13]. On the machines Bruce Dawson has used, the default tick is 15.625 ms (64 a second), so Sleep(1) called at a random moment wakes on the next tick, or the one after if the next is less than 1 ms away: anywhere from 1.0 to 16.625 ms later[14]. timeBeginPeriod(1) asks for 1 ms ticks. Since Windows 10 version 2004 the request no longer speeds up processes that didn't make one, and on Windows 11 it isn't guaranteed while the process's windows are minimized or hidden[15]. Since version 1803 a waitable timer created with CREATE_WAITABLE_TIMER_HIGH_RESOLUTION gets the same precision without a timeBeginPeriod call: the OS raises the tick rate itself, only for the part of the wait that needs it[16]. Linux wakes sleepers more precisely, but each thread has a timer slack, 50 µs by default, by which the kernel may wake it late in order to group wake-ups[17].

Worked example

A 60 fps cap with 4 ms of work leaves 12.67 ms to sleep. With 15.625 ms ticks and a frame that starts on a tick, Sleep(12) can't end at 16 ms, because no tick falls there. A thread that wakes on the first tick after its 12 ms have passed, the behavior Dawson describes, wakes at 31.25 ms. (Microsoft's documentation also lets a wait shorter than one tick end early, here at 15.625 ms. Either way the frame isn't 16.67 ms.) After a 31.25 ms frame the loop is behind schedule, so the next frame skips the sleep and runs short to catch up. The average over a second can still come out at 60 fps while no single frame lasts 16.67 ms, which the eye sees as judder. With 1 ms ticks the same sleep ends at 16 or 17 ms.

The usual cure is to sleep until a little before the deadline and then spin: loop on the clock, yielding, until the deadline passes. The spin costs CPU for a millisecond or two per frame and buys a wake-up within microseconds, though a busy machine can still preempt the spinning thread. In a game that presents with vsync, the swapchain's wait for the display does the pacing and the loop doesn't sleep at all; a cap matters for tools, menus, servers and vsync-off modes.
sleep, 15.625 ms ticks
sleep, 1 ms ticks
sleep, then spin
target
work 4 ms
Sleep, default 15.625 ms timeraverage 60.0 fps (target 60)frame times 3.5–31.3 mswithin 1 ms of target: 0%
Each bar is one frame's duration; the dashed line is the target. Switch the three ways of waiting, then change the target and the work per frame. The widget models the rule Dawson describes (a sleeping thread wakes on the first timer tick at or after the time it asked for) rather than measuring your machine. At 60 fps with 4 ms of work, 15.625 ms ticks still average 60 fps, but no frame lands within 1 ms of 16.67: about two thirds last exactly one tick, 15.625 ms, so the loop runs ahead of its schedule, and the rest swing between about 3.5 and 31 ms as it falls behind and catches up. 1 ms ticks put about 97% of frames within 1 ms. Sleep, then spin (sleep to 2 ms before the deadline, spin the rest) hits every deadline, at the cost of the spin.
A frame cap: sleep only, or sleep then spin, measured on your machine (complete program)
C++
Rust

// Capping the frame rate with a sleep, and with a sleep followed by a short spin.
// Prints how late each approach wakes up after its deadline on this machine.
#include <algorithm>
#include <chrono>
#include <cstdio>
#include <thread>
#include <vector>

using Clock = std::chrono::steady_clock;          // monotonic: unaffected by changes to the wall clock
using Seconds = std::chrono::duration<double>;

// Wait until the deadline: sleep while more than spinMargin remains, then spin on
// the clock for the rest, because the OS may wake a sleeping thread late.
void waitUntil(Clock::time_point deadline, Seconds spinMargin) {
    for (;;) {
        Seconds remaining = deadline - Clock::now();
        if (remaining <= Seconds::zero()) return;
        if (remaining > spinMargin)
            std::this_thread::sleep_for(remaining - spinMargin);
        else
            std::this_thread::yield();               // spin: stay awake, but let other threads run
    }
}

// Run frames of `work` capped at targetFps; return how late each frame's wait ended, in µs.
std::vector<double> runCapped(int frames, double targetFps, Seconds work, Seconds spinMargin) {
    const auto period = std::chrono::duration_cast<Clock::duration>(Seconds(1.0 / targetFps));
    const auto workTime = std::chrono::duration_cast<Clock::duration>(work);
    std::vector<double> lateMicros;
    Clock::time_point deadline = Clock::now();
    for (int frame = 0; frame < frames; ++frame) {
        Clock::time_point workEnd = Clock::now() + workTime;
        while (Clock::now() < workEnd) {}             // stand-in for update and render
        deadline += period;                           // a fixed schedule: one late frame doesn't shift the rest
        waitUntil(deadline, spinMargin);
        lateMicros.push_back(Seconds(Clock::now() - deadline).count() * 1e6);
    }
    return lateMicros;
}

void report(const char* label, std::vector<double> lateMicros) {
    std::sort(lateMicros.begin(), lateMicros.end());
    double mean = 0;
    for (double late : lateMicros) mean += late / lateMicros.size();
    std::printf("%-22s mean %7.1f us late, 99th percentile %7.1f us, worst %7.1f us\n", label, mean,
                lateMicros[lateMicros.size() * 99 / 100], lateMicros.back());
}

int main() {
    const int frames = 240;                           // two seconds at 120 fps
    const double targetFps = 120.0;                   // an 8.33 ms frame
    const Seconds work(0.003);                        // 3 ms of "game" per frame
    report("sleep only", runCapped(frames, targetFps, work, Seconds(0)));
    report("sleep, then spin 1 ms", runCapped(frames, targetFps, work, Seconds(0.001)));
}

Run it and the "sleep only" line reports how late your OS wakes a sleeping thread, averaged over 240 frames; the spin line usually lands within a few microseconds, with an occasional late frame when the scheduler runs something else on the core. On Windows the two tabs measure different sleeps: Rust's thread::sleep already uses a high-resolution waitable timer on Windows 10 version 1803 and later[18], while MSVC's sleep_for ends in Sleep with whole milliseconds[19], so only the C++ line shows the tick rounding. The deadline advances by a fixed period every frame instead of restarting from "now", so one late frame doesn't push every later frame back with it.

[Back to Platform & Window: the OS Event Loop](index.md)
