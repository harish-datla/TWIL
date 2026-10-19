# What's Next


Next in the series, The C++ Memory Model picks up from §15: what std::atomic and each memory_order promise, and why the instructions above are what they lower to. Beyond that, where to go from here:

    Read your own engine's hot paths. Pick the math, physics, animation, and renderer-glue libraries you ship. Build a release binary with debug info. objdump -d -M intel on Linux, dumpbin /disasm on MSVC, or the Compiler Explorer[1] "load my source" workflow. Look at what your hot inline functions actually compile to.
    Sit with Agner Fog's manuals.[5] The five PDFs at agner.org/optimize are the practitioner reference: manuals 1–4 cover optimizing C++, optimizing assembly, microarchitecture, and instruction tables; manual 5 is calling conventions. Free, and still updated (the microarchitecture manual was last revised in 2026).
    Run llvm-mca and uica on the loops you write. Both are free; both take a small region of assembly and predict throughput and port pressure. Use them before and after intrinsic refactors to confirm the change you intended is the change the model sees.
    Read a real ABI document. The function-calling chapter of the System V AMD64 ABI[6] gives a very different picture of "what a function call is" than the C language does. The classification of struct fields into register classes, aggregate passing, and the variadic-function rules are all worth a slow read.
    Pair with the Memory Model tutorial. It picks up where §15 stops: why each memory_order lowers to the instructions shown there, and what goes wrong when the choice is too weak.


[Back to x86-64 Assembly from Scratch](index.md)
