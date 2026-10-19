# The CPU You're Actually Programming For


Two loops with the same number of instructions can differ fourfold in speed on the same CPU, and nothing in the instruction list tells you which is which. The reason is that an assembly instruction is not what executes. A modern x86 core decodes each instruction into one or more μops, schedules those μops across multiple execution units in parallel, and retires them in program order at the back end[3]. The instruction stream you read is the contract; what runs is a heavily reordered, register-renamed, speculatively executed version of it. This section sketches that machine, enough to make sense of why some instructions are nearly free and others stall a whole pipeline.

A modern x86 pipeline, simplified to the parts you care about reading assembly for:

    Fetch. The CPU reads 16 to 32 bytes per cycle from the L1 instruction cache (32 KB on Skylake, Sunny Cove, Golden Cove, and Zen 2 through Zen 5; 64 KB on Zen 1 and Lion Cove[12]) into an aligned fetch window.
    Decode. Four decoders on Skylake and on Zen 3 and Zen 4, six on Golden Cove, eight on Lion Cove convert variable-length x86 instructions into fixed-format μops. Most simple instructions decode to a single μop; string ops, integer divide, and a handful of complex instructions decode to several.
    μop cache. Recent x86 cores keep a cache of already-decoded μops. A hot loop that fits in it (1.5K μops on Skylake, 4K on Golden Cove[12]) bypasses the legacy decoders entirely. AMD's equivalent on Zen 4 holds about 6.75K ops.
    Rename and dispatch. The renamer maps each architectural register named in the instruction to a physical register from a pool of a few hundred. This removes false dependencies: in imul rax, rbx followed later by mov rax, 5, the mov writes a fresh physical register, so it doesn't wait for the slow multiply to finish. True dependencies stay: add rax, 1; add rax, 2 still runs in order, because the second add reads the first one's result.
    Schedule and execute. The scheduler issues ready μops to a set of execution ports: 8 on Skylake, 10 on Sunny Cove, 12 on Golden Cove, and a comparable number on recent Zen cores[13]. A given port can start one μop per cycle. Ports are specialized; integer divide, store-data, branches, and vector multiply each live on a subset.
    Retire. A reorder buffer holds completed μops until every earlier μop has finished. Retirement happens in program order, at 4 μops per cycle on Skylake and 8 or more on the newest Intel and AMD cores[12]. Once an instruction's μops retire, its writes become architecturally visible; until then the CPU can roll them back (on a branch mispredict, an exception, or a memory-order violation).

The two practical consequences for reading assembly. First, an instruction's latency (cycles from issue to result available) is different from its throughput (number of times per cycle the CPU can issue it). A vmulps on Skylake has a latency of 4 cycles but a throughput of two per cycle[13].

Worked example

Eight vmulps on Skylake, each with 4 cycles of latency, two of which can start per cycle.

Chained, each multiply needing the previous result: 8 × 4 = 32 cycles.

Independent: two start per cycle, so all eight have started after 4 cycles, and the last result lands 4 cycles after that: about 8 cycles. Same instructions, roughly a 4× difference in finish time.

Second, the difference between a "fast" and "slow" implementation of the same algorithm at the assembly level is often not the instruction count; it is whether the instructions form a serial dependency chain that leaves most ports idle, or independent work the scheduler can spread across all of them.

The widget below schedules sixteen integer adds onto four ALU ports, once as a single dependent chain and once as four independent ones. §16 returns to this picture with a worked example:
Live · Port pressure scheduler
serial
parallel
Each row is an execution port and each block one add, with time running left to right. Toggle between serial and parallel and compare where the last block lands: cycle 16 against cycle 4. Sixteen add μops scheduled on a four-port integer ALU, modeled after Skylake's ports 0/1/5/6 (each accepts one integer add per cycle[13]). In serial mode each add waits for the previous one (one chain of dependent adds; 16 cycles). In parallel mode the same sixteen adds use four independent accumulators (one chain per port; 4 cycles). Same instructions, a 4× difference in finish time.

[Back to x86-64 Assembly from Scratch](index.md)
