# x86 vs ARM: The Latent Bug


A queue runs for months on PC and PS5, then corrupts data on the first Switch build. The most dangerous thing about lock-free code is that a wrong version often passes every test on your desktop. x86's memory model (its rules for how one thread's memory operations can appear reordered to another) is TSO, total store order, and it's strong: in hardware, a plain load is already an acquire and a plain store already a release[7][8]. So relaxed where you needed acquire/release usually goes unpunished, but it is not correct: the hardware supplies the ordering, and the compiler is still free to reorder your other loads and stores around a relaxed atomic.

ARM and POWER are weak, meaning the hardware may reorder independent loads and stores unless told not to: those reorderings really happen. The Cambridge lowering table makes it concrete: an acquire/release on x86 compiles to a plain MOV, while on AArch64, the 64-bit Arm instruction set, it needs LDAR/STLR instructions[6]. So a missing acquire/release can stay invisible on your dev box and corrupt state on a Switch, a phone, or Apple Silicon: the bug was always there, and the weaker hardware exposes it.

The PS5 and Xbox Series are x86, which hides this

The PS5 and Xbox Series are AMD Zen 2, the same x86-TSO model as a PC. A race that only appears there is timing exposing a bug that exists on x86 too; only genuinely weak targets (ARM) change the model itself. Test lock-free code on ARM hardware, or you are not testing the ordering.

Run the message-passing test: writer sets data = 42 then sets a flag; reader waits on the flag then reads data. Toggle the ordering and the hardware model:
relaxed
acquire / release
x86 (TSO)
ARM (weak)
Each trial runs the writer and the reader once; the counters show trials and stale reads, and the line below them shows when each store reached memory and when the reader read. Try all four combinations of ordering and hardware. With acquire/release the reader always sees 42; the stale-read counter stays at zero on either hardware. Switch to relaxed and pick x86: still zero, because TSO drains stores in program order (the demo models the hardware only; a real compiler could still reorder). Switch to ARM and the counter climbs, by about 3 trials in 10 in this model: the data write leaves the store buffer after the flag. (ARM can also reorder the reader's two loads; the demo models only the writer's side.) The per-trial timeline shows the order each event reached memory; drain times are randomized and illustrative, not measured silicon. That is why "it works on my machine" is not evidence here.
Check Your UnderstandingQ1 of 2

A queue is described as lock-free. What does that guarantee?
System-wide progress: in any execution, some operation always completes, though an individual thread can still starve.
No thread ever waits or blocks.
It never uses a mutex internally, which makes it faster.
0/2 correct

[Back to Lock-Free Queues and Memory Ordering](index.md)
