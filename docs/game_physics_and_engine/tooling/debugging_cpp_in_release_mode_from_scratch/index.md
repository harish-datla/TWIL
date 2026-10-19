# Debugging C++ in Release Mode from Scratch

Shipping builds are optimized, fast, and the builds that actually crash for players. This tutorial covers preserving useful symbols, reading optimized assembly, hardware watchpoints, sanitizers, crash dumps, heisenbugs, races, and game-specific debugging workflows.

Time: ~70 min  
Level: Intermediate to senior  
Prerequisites: C/C++, basic debugger use, x86-64 assembly, and the memory model.  
Phase: 13 · Tooling

## What you’ll be able to do

- Produce full-speed shipping builds that still resolve crashes to functions and source lines.
- Read optimized x86-64 assembly and identify stack frames, arguments, library calls, and virtual calls.
- Catch memory corruption with hardware watchpoints and automate debugger investigation.
- Choose sanitizers for memory, undefined-behavior, and data-race bugs.
- Triage Windows minidumps and Linux core files.

## Sections

- [The Skill That Actually Ships](01-the-skill-that-actually-ships.md)
- [What the Binary Still Knows](02-what-the-binary-still-knows.md)
- [Building for Debuggability](03-building-for-debuggability.md)
- [The Debugger as a Data Tool](04-the-debugger-as-a-data-tool.md)
- [Hardware Watchpoints](05-hardware-watchpoints.md)
- [Reading Optimized Assembly](06-reading-optimized-assembly.md)
- [Fingerprinting Library Code](07-fingerprinting-library-code.md)
- [Sanitizers](08-sanitizers.md)
- [Crash Dumps and Post-Mortem Inspection](09-crash-dumps-and-post-mortem-inspection.md)
- [Heisenbugs and Undefined Behavior](10-heisenbugs-and-undefined-behavior.md)
- [Race Conditions and TSan](11-race-conditions-and-tsan.md)
- [Game-Specific Debugging](12-game-specific-debugging.md)
- [Pitfalls](13-pitfalls.md)
- [What’s Next](14-whats-next.md)
- [Sources & Further Reading](15-sources-and-further-reading.md)
