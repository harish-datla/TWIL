# A tiny profiler

You want every thread's scopes on one timeline today, without integrating a profiler library first. The pieces fit in under 150 lines: a scoped timer, a buffer per thread, and an exporter that writes the Chrome Trace Event Format, a JSON array of events that Chrome's trace viewer (chrome://tracing) reads[13]. The Perfetto UI at ui.perfetto.dev opens the same file directly and draws a named track per thread[14]. The Bevy engine works the same way: its built-in spans start a timer when entered and stop it when dropped, and its trace_chrome feature writes this JSON for Perfetto[15].

    Recording takes no lock. Each thread appends zones to its own thread_local buffer. The lock is taken once per thread, when the thread finishes and hands its buffer to the profiler.
    A zone is 24 bytes. A pointer to a string literal, a start time and a duration (32 bytes in Rust, where a &str also carries its length). After the up-front reserve, recording a zone doesn't allocate.
    The scope does the timing. The constructor reads the clock and the destructor reads it again, so an early return or an exception still closes the zone. That's RAII; in Rust it's Drop.
    The export is two event types. One "X" (complete) event per zone with a start ts and a dur, both in microseconds, and one "M" (metadata) event per thread that names its track[13].

A complete scoped-zone profiler that writes a Chrome trace
C++20
Rust 2021

// tiny_profiler.cpp: scoped zones on every thread, written as a Chrome trace.
// Build: g++ -std=c++20 -O2 -pthread tiny_profiler.cpp   Open trace.json at ui.perfetto.dev
#include <chrono>
#include <cmath>
#include <cstdint>
#include <cstdio>
#include <mutex>
#include <string>
#include <thread>
#include <utility>
#include <vector>

using ProfileClock = std::chrono::steady_clock;          // monotonic: never jumps backwards

struct ZoneEvent {                                       // one finished zone, 24 bytes
    const char* name;                                    // a string literal, so nothing is copied
    std::int64_t startNs;                                // nanoseconds since the profiler started
    std::int64_t durationNs;
};

struct ThreadBuffer {
    std::string threadName;
    std::vector<ZoneEvent> events;
};

// Each thread records into its own buffer, so recording a zone takes no lock.
thread_local ThreadBuffer currentThreadBuffer;

class Profiler {
public:
    static Profiler& instance() {
        static Profiler profiler;                        // thread-safe initialization since C++11
        return profiler;
    }

    std::int64_t nowNs() const {
        return std::chrono::duration_cast<std::chrono::nanoseconds>(ProfileClock::now() - epoch).count();
    }

    // Called once per thread when it finishes: the only place that takes the lock.
    void flushThisThread() {
        std::lock_guard<std::mutex> lock(finishedMutex);
        finishedThreads.push_back(std::move(currentThreadBuffer));
        currentThreadBuffer = ThreadBuffer{};
    }

    // Chrome Trace Event Format: a JSON array that Perfetto and chrome://tracing open.
    bool writeChromeTrace(const char* path) {
        std::FILE* file = std::fopen(path, "w");
        if (file == nullptr) return false;
        std::lock_guard<std::mutex> lock(finishedMutex);
        std::fputs("[\n", file);
        for (std::size_t threadIndex = 0; threadIndex < finishedThreads.size(); ++threadIndex) {
            const ThreadBuffer& buffer = finishedThreads[threadIndex];
            // "M" metadata event: names this thread's track in the viewer.
            std::fprintf(file, "%s{\"name\":\"thread_name\",\"ph\":\"M\",\"pid\":1,\"tid\":%zu,\"args\":{\"name\":\"%s\"}}",
                         threadIndex == 0 ? "" : ",\n", threadIndex, buffer.threadName.c_str());
            for (const ZoneEvent& zone : buffer.events) {
                // "X" complete event: ts and dur are microseconds, so three decimals keep whole nanoseconds.
                std::fprintf(file, ",\n{\"name\":\"%s\",\"ph\":\"X\",\"pid\":1,\"tid\":%zu,\"ts\":%.3f,\"dur\":%.3f}",
                             zone.name, threadIndex, static_cast<double>(zone.startNs) / 1000.0,
                             static_cast<double>(zone.durationNs) / 1000.0);
            }
        }
        std::fputs("\n]\n", file);
        return std::fclose(file) == 0;
    }

private:
    ProfileClock::time_point epoch = ProfileClock::now();
    std::mutex finishedMutex;
    std::vector<ThreadBuffer> finishedThreads;
};

void nameThisThread(const char* threadName) {
    currentThreadBuffer.threadName = threadName;
    currentThreadBuffer.events.reserve(1 << 16);         // grow once up front, not in the middle of a frame
}

class ScopedZone {                                       // RAII: the destructor closes the zone
public:
    explicit ScopedZone(const char* zoneName) : name(zoneName), startNs(Profiler::instance().nowNs()) {}
    ~ScopedZone() {
        const std::int64_t endNs = Profiler::instance().nowNs();
        currentThreadBuffer.events.push_back({name, startNs, endNs - startNs});
    }
    ScopedZone(const ScopedZone&) = delete;
    ScopedZone& operator=(const ScopedZone&) = delete;

private:
    const char* name;
    std::int64_t startNs;
};

// Paste in two steps so __LINE__ expands first (zone_118, say), not the literal zone___LINE__,
// which would collide when two zones share a scope.
#define ZONE_CONCAT_INNER(a, b) a##b
#define ZONE_CONCAT(a, b) ZONE_CONCAT_INNER(a, b)
#define ZONE_SCOPED(zoneName) ScopedZone ZONE_CONCAT(zone_, __LINE__)(zoneName)

// A volatile store keeps the work from being optimized away; thread_local so the threads don't race on it.
thread_local volatile double workSink = 0.0;

void busyWork(int iterations) {
    double total = 0.0;
    for (int i = 0; i < iterations; ++i) total += std::sqrt(static_cast<double>(i));
    workSink = total;
}

void updatePhysics() {
    ZONE_SCOPED("updatePhysics");
    { ZONE_SCOPED("broadPhase"); busyWork(200000); }
    { ZONE_SCOPED("solveContacts"); busyWork(600000); }
}

int main() {
    nameThisThread("main");
    std::thread streamingThread([] {
        nameThisThread("streaming");
        for (int request = 0; request < 3; ++request) {
            ZONE_SCOPED("decompressChunk");
            busyWork(900000);
        }
        Profiler::instance().flushThisThread();
    });
    for (int frame = 0; frame < 3; ++frame) {
        ZONE_SCOPED("frame");
        updatePhysics();
        { ZONE_SCOPED("render"); busyWork(500000); }
    }
    streamingThread.join();
    Profiler::instance().flushThisThread();              // main's zones have all closed by now
    if (!Profiler::instance().writeChromeTrace("trace.json")) {
        std::fputs("could not write trace.json\n", stderr);
        return 1;
    }
    std::puts("wrote trace.json");
    return 0;
}

Run either program and it writes trace.json; drop the file on ui.perfetto.dev and you get a main track with three frame zones, each holding updatePhysics (with broadPhase and solveContacts inside) and render, and a streaming track with three decompressChunk zones running alongside.
What's intentionally missing

    A memory bound. Buffers grow until their thread finishes. A long session needs a ring buffer, or a background thread that streams events to disk or over a socket, which is how Tracy works.
    Live viewing. Nothing leaves a thread until it calls flushThisThread, so you can't inspect a frame while the game runs, and a thread that exits without flushing loses its zones.
    Frame markers, GPU zones, locks, allocations and sampling. A real engine profiler records all of them on the same timeline.
    JSON escaping. Zone and thread names go into the file as-is, so a name containing a quote breaks the trace.
    A cheaper clock. Each zone reads steady_clock twice. Tracy reads the CPU's timestamp counter directly (the rdtsc instruction on x86), which gives it single-digit-nanosecond resolution[9].

Every zone costs time on the thread it measures, so the profiler changes what it measures. Tracy's manual measures its own cost on a texture-compression run spread over 12 threads: 37.7 ms of extra wall-clock time across 16.8 million zones, 2.25 ns per zone[9]. The 12 threads shared that cost, so if they stayed busy throughout, each zone cost about 12 × 2.25 ≈ 27 ns on the thread that recorded it. A zone built from two steady_clock reads and a push_back, like the one above, costs more than Tracy's tuned path, so the same arithmetic applies with a bigger number.

Worked example

Say a zone costs 30 ns and you wrap the per-particle update, 50 ns of work, called 20,000 times a frame. The particle loop really takes 20,000 × 50 ns = 1.0 ms. The zones add 20,000 × 30 ns = 0.6 ms, so the enclosing zone reports 1.6 ms and the particle system looks 60% slower than it is. One zone around the whole loop costs 30 ns once and reports 1.0 ms. Put zones around systems and loops, and leave the per-item work to the sampler.
