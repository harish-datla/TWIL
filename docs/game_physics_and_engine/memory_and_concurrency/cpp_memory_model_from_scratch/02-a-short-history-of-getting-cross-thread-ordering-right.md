# A Short History of Getting Cross-Thread Ordering Right


Why does C++ have six memory orderings, and why does old engine code use volatile for thread flags? Because the C++ memory model didn't arrive fully formed in 2011. It grew out of three decades of work on what hardware and compilers actually guarantee, and code written before it relied on whatever the compiler and CPU of the day happened to do. Its shape makes more sense next to what came before it.
Timeline: from Lamport (1979) to C++26

The short version: the "threads take turns" model, called sequential consistency, dates from 1979; precise models of what x86 and ARM hardware really do arrived around 2010; the C++ standard got its memory model in 2011; and the repairs that made it match weak hardware came between 2013 and 2018. Plenty of production code is still written against the 1979 picture rather than what the hardware and the standard actually promise.

[Back to The C++ Memory Model from Scratch](index.md)
