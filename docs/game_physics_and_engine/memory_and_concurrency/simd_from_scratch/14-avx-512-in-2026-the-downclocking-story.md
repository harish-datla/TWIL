# AVX-512 in 2026: The Downclocking Story


On some older Intel servers, switching one kernel to AVX-512 made the scalar code around it slower. Game code rarely uses AVX-512, first because most Intel client chips of the last few years don't have it, and second because of its reputation for lowering clock speed. The reputation comes from Skylake-SP: "AVX-512 instructions drop the CPU's clock, the clock drop eats the speedup, so don't use them." That was a fair summary of the first generation and has been less true with each one since.

What actually happened, generation by generation:

    Skylake-SP (Xeon Scalable 1st gen, 2017) and its Xeon W siblings. Frequency licenses: L0 for scalar and light vector code, L1 for heavy AVX2 or light AVX-512, L2 for heavy (floating-point and multiply) AVX-512. Travis Downs's test chip, a Xeon W-2104, runs at 3.2, 2.8, and 2.4 GHz in L0, L1, and L2; he measured a stall of roughly 8 to 20 µs at each voltage transition during which the core dispatches at a quarter of its normal rate, and about 680 µs after the last wide instruction before the core returns to L0[26]. Because the clock applies to the whole core, a few AVX-512 kernels scattered through scalar code slow the scalar code down too.
    Cascade Lake (2019). The same license scheme.
    Ice Lake (client 2019, server 2021). Much smaller penalties. On an Ice Lake laptop chip, Downs measured heavy AVX-512 at 3.6 GHz against 3.7 GHz for scalar code with one core active, and no license-based reduction at all with more cores active[26]. Phoronix measured the Ice Lake Xeon Platinum 8380's peak clock dropping by about 175 MHz under AVX-512-heavy benchmarks[36].
    Sapphire Rapids (2023). In the same Phoronix tests, peak frequency was similar with and without AVX-512[36].
    Intel client (Alder Lake / Raptor Lake / Meteor Lake / Arrow Lake, 2021–2024). AVX-512 is disabled or absent because the E-cores (the small, low-power cores in Intel's hybrid chips) lack the unit; this is the source of "modern Intel desktops don't have AVX-512" in 2026.
    AMD Zen 4 (2022). Executes each 512-bit operation as two halves on its 256-bit datapaths, so 512-bit throughput per instruction is half of a two-FMA-unit Intel server core's. There is no AVX-512-specific clock offset; Phoronix saw no AVX-512 downclocking on Genoa[36][37].
    AMD Zen 5 (2024). Full 512-bit datapaths on desktop and server parts; the Strix Point laptop chips keep 256-bit datapaths[37]. There are no fixed AVX-512 frequency offsets, though code that combines 512-bit loads with FMAs makes the core back off its clock gradually over a few milliseconds, as any power-hungry code does[38].

Worked example

On Downs's Xeon W-2104, L2 runs at 2.4 GHz against 3.2 GHz in L0: 75% of the clock, so code there takes a third longer.

A frame that runs heavy AVX-512 code at least every 0.5 ms never goes 680 µs without it, so the core never returns to L0. It stays at 2.4 GHz for the whole frame, so the scalar systems between the kernels run a third longer as well.

The practical advice in 2026:

    On current servers (Xeon Ice Lake or later, EPYC Zen 4 or later), use AVX-512 where it measures faster.
    On consumer Intel from Alder Lake through Arrow Lake, AVX-512 isn't available; -march=x86-64-v3 (AVX2 / FMA / BMI2) is the target for code that has to run on those machines.
    On consumer AMD from Zen 4 onward, AVX-512 is available with no AVX-512-specific clock penalty.
    For shipping game binaries, the usual pattern is multi-versioning: compile two or three versions of the hot kernels for different instruction sets and pick one at startup. GCC and Clang can generate the variants and the dispatcher from target / target_clones attributes (function multiversioning)[27]; MSVC has no equivalent, so Windows builds typically compile separate files per instruction set and fill a function-pointer table after checking CPUID.

The Skylake-SP-era advice to avoid AVX-512 for frequency reasons describes hardware few players own. The current advice: use AVX-512 where it's available and faster, fall back to AVX2 where it isn't, and dispatch at startup.

[Back to SIMD from Scratch](index.md)
