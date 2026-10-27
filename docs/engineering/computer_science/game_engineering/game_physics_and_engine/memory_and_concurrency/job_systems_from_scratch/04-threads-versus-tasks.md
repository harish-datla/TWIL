# Threads versus Tasks


A first attempt at parallelism often spawns a thread per piece of work: one for each enemy's AI, one for each chunk of physics. On an 8-core machine that's hundreds of threads fighting over eight cores. The mistake is conflating two separate ideas.

A thread is an execution context: a stack, a program counter, and a slot in the operating system's scheduler. Creating and joining one on Linux costs on the order of 10 microseconds (9 µs on an Intel Skylake server and 20 µs on AMD Rome in Lemire's measurement[15]). Each one reserves address space for its stack (1 MiB by default on Windows, 8 MiB with a typical Linux default), and once you have more threads than cores, the kernel has to context-switch between them, which costs on the order of a microsecond per switch, more once you count the cache pollution that follows. So you want few threads.

A task (or job) is a unit of work: a function and its inputs. Creating one costs little more than filling in a small struct, so you can afford many tasks.
The rule

One worker thread per physical core (or one per logical core minus one, leaving one for the OS, depending on platform). Then divide the actual work into hundreds or thousands of tasks that those workers chew through.

Unity's manual states it directly: the job system "ensures that there are only enough threads to match the capacity of the CPU cores"[8]. Naughty Dog runs six worker threads on PS4, pinned one per core, and then puts a hundred and sixty fibers on top of them to hold the work[2]. Both engines are following the same rule.

In our scheduler, the worker count follows that rule. It says how many threads to start on the machine the game finds itself on:
eq. 1 · worker count workers = max(1, Ncores − 1)

In words: one worker per core, minus one core kept for the main thread and the OS, and never fewer than one worker. On an 8-core machine that's 7 worker threads plus the main thread, so 8 threads total but only 7 of them are dedicated to jobs.

Worked example

On a 16-thread desktop CPU (8 cores, 2 hardware threads each), hardware_concurrency() returns 16, so eq. 1 gives 16 − 1 = 15 workers. On a single-core device it gives max(1, 0) = 1, which is why the max is there.

Memory is the other difference between threads and tasks. Fifteen workers reserve 15 × 1 MiB = 15 MiB of stack address space at Windows' default. A thousand jobs at Molecular Matters' 64 bytes each take 64,000 bytes, less than one thread's stack.

Once you separate the two concepts, a few other things become obvious. Tasks are cheap to make, so you should make a lot of them: granularity matters, and we'll quantify it in §7. Tasks don't need their own stack: they borrow the worker's stack while they run, so a task can fit in a few dozen bytes (Molecular Matters sizes its jobs to one 64-byte cache line[7]) instead of a megabyte.
Check Your Understanding Q1 of 2

Why would a job system create only N − 1 worker threads on an N-core machine instead of N?
Leaving a core free for the OS and the main thread reduces context-switch pressure and makes it less likely the OS preempts a worker when it wants to run something else.
It uses less memory.
Most CPUs hit a thermal limit before all cores are utilized.
0/2 correct

[Back to Job Systems from Scratch](index.md)
