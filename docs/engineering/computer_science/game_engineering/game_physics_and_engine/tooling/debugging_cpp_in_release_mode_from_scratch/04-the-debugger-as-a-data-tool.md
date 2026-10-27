# The debugger as a data tool

A navmesh query goes wrong somewhere past its 10,000th node, and only on one level. Clicking Continue ten thousand times isn't a plan. Most people picture a debugger as a UI: set a breakpoint, hit it, look at the locals. That covers a small part of what it can do. A debugger is a programmable inspection tool attached to a running process, with full access to memory, registers, threads, system calls and signals. Many of the bugs that don't fall out of "stop and look around" give way once you script the debugger to collect data over many iterations and then read the result.

The four classes of tactic worth knowing, all available in GDB, LLDB, and WinDbg in some form:
Conditional and counted breakpoints

"Stop here, but only when i > 10000 and node->parent == nullptr." On GDB this is a conditional breakpoint[16]: break NavMesh.cpp:402 if i > 10000 && node->parent == nullptr. The cost is a trap into the debugger, a condition evaluation and a resume on every hit, which on a hot path makes the program crawl. The usual fix is to move the condition into the program for the duration of the hunt: if (i > 10000 && !node->parent) __builtin_trap(); (__debugbreak() on MSVC), rebuild, and let the hardware trap only when it matters. With a remote stub, GDB's set breakpoint condition-evaluation target has gdbserver evaluate the condition itself, which saves the round-trip to GDB but not the trap[16].

"Stop here on the 1024th hit." Ignore counts: ignore 1 1023 on GDB, breakpoint modify --ignore-count 1023 1 on LLDB. Useful for finding the iteration where an invariant first breaks: ignore until just before, single-step from there.
Tracepoints and breakpoint commands

"Don't stop, just log." GDB's dprintf (a breakpoint that prints and continues), a breakpoint with a commands ... continue block, a Visual Studio tracepoint, or a WinDbg bp whose command string ends in gc prints values to a log and lets the program run on. Log the call site, the argument, a timestamp, and a small fingerprint of the relevant state; run the failing scenario; read the log offline. It often replaces printf-debugging because it doesn't require a recompile, which matters when a link takes minutes or the bug only reproduces in the build already running. It still stops the thread for each hit, so it perturbs timing more than a compiled-in log line.
Reverse and replay debugging

rr, originally developed at Mozilla and now maintained at rr-debugger/rr, records the system calls and non-deterministic events of a Linux process and replays them deterministically inside GDB[17]. Inside the replay you can step backwards, set breakpoints in the future and run forward, and re-run the same buggy execution as many times as you need. rr runs all of the recorded threads on a single core, so heavily parallel programs slow down the most: the rr paper measured recording slowdowns of 1.49× to 1.79× on four real-world workloads and 7.85× on a parallel make, with replay usually no slower than recording[17]. WinDbg has Microsoft's Time Travel Debugging for the same workflow on Windows[18]. The class of bug rr and TTD solve faster than anything else: "the value of this pointer changed and I don't know who changed it". Run backwards from the wrong value to the assignment.

Worked example

A bug takes 30 seconds of play to reproduce. At the rr paper's 1.49× to 1.79× recording slowdowns, recording it takes about 45 to 54 seconds; at the 7.85× it measured for a parallel make, about 4 minutes. You pay that once. Every replay after it runs the identical execution, so the pointer gets the same wrong value at the same instruction every time, and a watchpoint plus reverse-continue lands on the store that wrote it.
Scripted post-mortem inspection

The Python APIs of GDB and LLDB, and the JavaScript dx engine in WinDbg, let you walk arbitrary structures programmatically. A short script that walks every entity in the ECS and reports the ones with a specific component-mask flag set answers the question in one pass. Scripting the debugger is the right call when the question is "across all current data, which N satisfy P," and the alternative is clicking through ten thousand entries by hand.
Tip · Save the session

A debugging session that found a bug is also a regression test for whether you fixed it. Save the breakpoint definitions, the scripted commands, and the conditional expressions to a .gdbinit, .lldbinit, or WinDbg script. Re-running the same script against the patched binary should now reach the end without tripping the assertion or logging the off-by-one. The script becomes documentation that the next person on the bug can re-execute.
