# From where to why

The profile says the contact solver takes 1.4 ms. It doesn't say why. The CPU can tell you: its performance counters count cycles, instructions retired, cache misses and branch mispredictions while the code runs. perf stat prints the counts with IPC (instructions per cycle) next to them[17].

Raw counts are hard to weigh against each other, so Intel's top-down method sorts the core's pipeline slots, its capacity to issue work each cycle, into four bins. Retiring is useful work. Bad Speculation is work thrown away, mostly after mispredicted branches. Front-End Bound means the core couldn't fetch and decode instructions fast enough. Back-End Bound means decoded work was waiting on the back end, and it splits into Memory Bound (cache misses and memory accesses) and Core Bound (execution units used poorly, such as several divides in a row competing for the divide units)[18]. On CPUs that support it, perf stat --topdown prints the four top-level numbers[17].

Worked example

Say the solver runs 1.4 ms on a 3.0 GHz core: 4.2 million cycles. perf stat reports 1.7 million instructions, an IPC of 0.4 on a core that can retire several per cycle, and top-down puts most of the slots in Memory Bound. Fewer instructions won't help much; the core is waiting on memory. The usual fix is data layout, contacts stored contiguously and walked in order, which the Memory Allocators and ECS pages cover.
