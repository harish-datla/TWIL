# What's next

Where to go from here:

    Next in the series: Tooling & Profiling, the in-engine side of this work: a dev UI to inspect and tweak the game live, CPU and GPU profilers, and a frame debugger.
    The x86-64 Assembly tutorial, for the ISA-level vocabulary §6 used freely.
    The Memory Model tutorial, for the foundation under §11 on data races.
    Read the DWARF 5 spec[7]. §6.2 (the line-number program) and §6.4 (call frame information) are the parts that pay off most quickly.
    Brendan Gregg's Systems Performance[44] for the long-form treatment of perf, eBPF, and the production-debugging tooling on Linux.
    John Robbins' Debugging Applications for Microsoft .NET and Microsoft Windows[45]. Covers WinDbg, minidumps, and the SOS extension at length. It dates from the Windows XP era, but the native-debugging chapters still apply.
    Try rr on a real bug. The on-ramp is small (rr record ./your_program, then rr replay), and the workflow is hard to give up once you've used it for a "who set this pointer" hunt.
    Set up a symbol server for your team. The friction of "I can't symbolicate my crash" is a common reason dump-collection pipelines go unused; a working symbol server removes it.

