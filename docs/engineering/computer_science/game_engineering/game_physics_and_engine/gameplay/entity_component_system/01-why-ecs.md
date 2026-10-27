# Why ECS

A shooter updates fifty thousand objects every frame, and most of that time goes to fetching each one from a different place in memory. Game objects in a shipping engine carry a variable set of behaviors: this one has a transform and a mesh, that one has a transform, a mesh, a rigid body, and an AI controller. The classical OOP approach models this with inheritance. Two decades of shipped games have shown what deep inheritance hierarchies cost at scale: diamond problems, fat base classes, and scattered memory layouts that cost real frame time. ECS is one widely adopted answer.

The core proposition: separate identity from data from behavior. An entity is a lightweight ID. A component is a plain data struct attached to that ID. A system is a function that runs over all entities matching a component query. There is no inheritance and no virtual dispatch: components live in flat, typed arrays that systems walk sequentially, so the CPU's prefetcher sees a predictable stride, and the scheduler sees declared read and write sets it can use to run systems in parallel.

The results show up in frame time, and the cleanest published measurement shows which cost dominates.

Worked example

Tony Albrecht, at Sony's European R&D group, timed 11,111 objects in a five-level scene tree, transformed and hierarchically culled every frame, with an empty render call so only the traversal counted. At the starting point the profiler showed 50,421 branch mispredictions at roughly 23 cycles each (about 0.36 ms) against 36,345 L2 cache misses at roughly 400 cycles each (about 4.54 ms): memory stalls cost more than twelve times what mispredicted branches did. Restructuring the loop from the textbook object hierarchy toward flat, contiguous data took it from 19.6 ms to 3.3 ms on PS3-era hardware, in three steps (19.6, 12.9, 4.8, 3.3 ms), about six times faster with the logic unchanged[4].
