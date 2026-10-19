# Where to Go from Here


Next in the series: SIMD. A job system spreads work across cores; SIMD makes each core process 4, 8 or 16 values per instruction. Engines use both on the same hot loops, and the layout that splits cleanly into jobs, contiguous arrays of independent elements, is also the layout SIMD wants.

You now have a code-level picture of what a job system is and why most modern engines have one. The other next step is reading other people's code: shipped schedulers make different trade-offs, and reading a few builds the instincts.
Read these libraries

    EnkiTS: Doug Binks's zlib-licensed task scheduler in plain C++[37]. Compact, with no fibers; a readable small example of a production scheduler.
    Marl: Google's hybrid thread + fiber scheduler[22]. Hand-written fiber switches for eight CPU architectures, plus work stealing.
    FiberTaskingLib: an explicit reimplementation of the Naughty Dog design[38]. Reading this is the easiest way to understand the Gyrling talk.
    Intel GTS: Intel's Games Task Scheduler[39]. Optimized for game workloads, with detailed documentation of design choices.
    NVIDIA stdexec: the reference C++26 senders/receivers implementation[29].

Read these papers

    Blumofe and Leiserson, Scheduling Multithreaded Computations by Work Stealing[20]. The mathematical foundation.
    Chase and Lev, Dynamic Circular Work-Stealing Deque[6]. The data structure.
    Lê, Pop, Cohen, Zappa Nardelli, Correct and Efficient Work-Stealing for Weak Memory Models[18]. The version you'd actually ship.
    Frigo, Leiserson, Randall, The Implementation of the Cilk-5 Multithreaded Language[19]. The work-first principle.

Talks

    Gyrling, Parallelizing the Naughty Dog Engine Using Fibers, GDC 2015[2]. The canonical AAA reference.
    Tatarchuk, Destiny's Multithreaded Rendering Architecture, GDC 2015[3]. The companion case study.
    Parent, Better Code: Concurrency, code::dive 2016[40]. The C++ futures-and-channels mindset.
    Stevens, How to Benefit from Multithreading in Your Unreal Engine Projects, Unreal Fest Gold Coast 2024[28]. The recent Epic perspective.

The final exam

Five questions covering the whole tutorial. If you can answer all five without scrolling back, you've got the fundamentals.
Check Your Understanding Q1 of 5

You’re building a 60 fps game on an 8-core machine. Your renderer takes 12 ms on its dedicated thread; every other subsystem takes 4 ms on its dedicated thread. Why is moving to a job system the obvious next step?
Because job systems are newer.
The frame is bottlenecked on the renderer’s single thread; the other seven cores are mostly idle. Splitting the renderer’s work into many small jobs lets all eight cores chip in.
Job systems use less memory.
0/5 correct

[Back to Job Systems from Scratch](index.md)
