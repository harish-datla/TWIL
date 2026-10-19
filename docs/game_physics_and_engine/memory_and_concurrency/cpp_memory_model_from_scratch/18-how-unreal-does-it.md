# How Unreal Does It


Open an older Unreal Engine subsystem and you'll find FThreadSafeCounter and FPlatformAtomics calls where this page uses std::atomic. They're Unreal's own atomic layer, older than its adoption of std::atomic:

    FPlatformAtomics. Per-platform functions (InterlockedIncrement, InterlockedAdd, InterlockedExchange, InterlockedCompareExchange, with 64- and 128-bit versions on platforms that support them) that map to each platform's locked instructions or compiler intrinsics[31][41].
    FThreadSafeCounter and FThreadSafeCounter64. Counter classes that wrap a volatile integer and route every operation through FPlatformAtomics[41]. They take no ordering argument.
    TAtomic<T>. A typed wrapper. Epic's coding standard now tells new code to use std::atomic, describing TAtomic as only partially implemented and not something Epic intends to maintain[42].

Applied to an Unreal codebase, this page comes down to a few habits:

    Write new lock-free code with std::atomic and explicit orderings, as the coding standard asks, so a statistics counter can be relaxed and a publication flag release/acquire.
    Use compare_exchange_weak inside retry loops and compare_exchange_strong for one-shot attempts (§13).
    Pad the two indices of every SPSC ring (render-to-RHI commands, audio events, profiler scopes) onto separate cache lines (§11).
    For a double-width CAS where the toolchain's 16-byte std::atomic isn't lock-free, FPlatformAtomics::InterlockedCompareExchange128 exposes the hardware instruction on platforms that have it.


[Back to The C++ Memory Model from Scratch](index.md)
