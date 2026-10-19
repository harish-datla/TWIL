# Crash dumps and post-mortem inspection

A player's game crashed overnight and the crash reporter uploaded a dump file. That file, plus the symbols for that exact build, is all you get. A crash dump is a snapshot of a process's address space at the moment it faulted. With matching debug info, the snapshot is a full debugger session you can re-attach to as many times as you need. With no debug info, it is a wall of hex. The format and the tooling differ by platform, but the workflow is the same.
Windows: minidumps

MiniDumpWriteDump writes a structured snapshot to a .dmp file[32]. The flags chosen at write time decide what's in it. MiniDumpNormal is the smallest: registers, stack of every thread, list of loaded modules, system info. MiniDumpWithFullMemory is the largest (the entire process's committed memory): every heap, every mapped file, every thread stack. The middle flags (WithDataSegs, WithProcessThreadData, WithIndirectlyReferencedMemory) are the practical compromise for shipping titles that need to investigate without uploading gigabytes per crash.

Open a minidump in WinDbg with windbg -z game.dmp, point it at the matching PDB and source server with .sympath and .srcpath, and the standard commands work as if the process were live: k for the stack, ~* k for every thread's stack, !analyze -v for the heuristic root cause, dt for type-directed memory inspection, !heap -p -a addr for the allocation history of an address with Page Heap on. The Microsoft public symbol server (https://msdl.microsoft.com/download/symbols) provides public PDBs for Windows system binaries; configure it once and your stack traces include named frames inside kernel32, user32, d3d11[11].
Linux: core dumps

The kernel writes a core file when a process is killed by a signal that's set up to dump (SIGSEGV, SIGABRT, SIGBUS, etc.) and ulimit -c permits it. The path pattern lives in /proc/sys/kernel/core_pattern and on distros that install systemd-coredump (Fedora and Arch by default) pipes the dump to it[33]. coredumpctl debug PID drops you into GDB attached to the snapshot and the binary that produced it; GDB then finds separate debug files by the binary's build ID. Ubuntu defaults to apport instead, with the same idea and a different command-line shape.

The same workflow without systemd-coredump: gdb game core. Point set debug-file-directory at where your .debug files live; when the core came from another machine, point set sysroot at a copy of its shared libraries; bt to walk the stack; info threads to list every thread; thread apply all bt to walk every stack at once.
What to look at first

A consistent triage order, roughly platform-independent:

    Faulting instruction and address. What kind of fault was it (read, write, execute) and at what address. A SIGSEGV with an address near zero is a null deref, usually of a field at a small offset. An address with all the high bits set (e.g., 0xfffffffffffffff8) is often a small negative offset from a null pointer. A fault address that is itself a fill pattern (0xfeeefeee... from the Windows debug heap's freed blocks, 0xdddddddd..., 0xcdcdcdcd..., 0xdeadbeef) means a pointer was loaded from memory the runtime had filled with that pattern: usually freed or uninitialized memory.
    The faulting thread's stack. Count frames; identify the topmost one in code you own. The crash is often caused by the wrong arguments coming into one of your functions from a library call.
    Other threads' stacks. If the bug is a race, the corruptor is on a different thread. Look for threads inside memcpy or memset with their destination overlapping your structure. For a hang rather than a crash, look for threads each waiting on a lock another one holds; that cycle is a deadlock.
    The argument registers at the crash. The first six (System V) or four (Windows) integer registers are the arguments to whatever function was being called. A SIGSEGV on the first instruction of a function, when that instruction dereferences rdi and rdi == 0, means someone called you with a null first argument on Linux.
    The address of the corruption. If a sanitizer is installed (or Page Heap, or GWP-ASan) and reported the corruption, use its allocation/free history. Otherwise use a hardware watchpoint on the next reproduction.

Worked example

A Windows dump reports an access violation reading address 0x0000000000000018, at mov rax, [rcx+0x18], with rcx = 0. On Windows x64, rcx carries the first argument, which for a member function is this: the method was called on a null object, and 0x18 is the field it was reading, 24 bytes in (the fourth field if they're all 8 bytes). The question is who passed the null, so walk up the stack to the first frame you own. A fault address of 0xfffffffffffffff8 is 264 − 8, which is 0 − 8 wrapped around: a load at offset −8 from a null pointer.
