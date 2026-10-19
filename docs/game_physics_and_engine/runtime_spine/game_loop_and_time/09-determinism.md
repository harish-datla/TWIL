# Determinism


A fighting game records only the players' inputs for its replays. Played back on the same build and machine, a replay matches. On a teammate's PC built with a different compiler, a hit that landed now misses. A fixed timestep is the usual foundation for a reproducible simulation, since every run takes the same-sized steps, but it is not enough on its own.

Three sources of divergence survive a fixed step. Floating-point results differ across compilers, optimization levels, architectures, and fast-math settings (compiler modes that let the optimizer reorder and fuse float math for speed), so the same code can produce slightly different numbers on different toolchains. Float addition is not associative, so changing the order operations are summed changes the result. And input timing or sampling can differ run to run. Nystrom puts the practical consequence plainly: with variable steps "the same bullet will end up in different places on their machines"[2]; a fixed step removes that particular variable but not the float ones.

Worked example

Three forces act on a body along one axis: a = 100,000,000, b = −100,000,000 and c = 1, stored as 32-bit floats. Summed as (a + b) + c the total is 1. Summed as a + (b + c), the inner sum rounds back to −100,000,000, because floats near 108 are 8 apart, and the total is 0. If a job system adds the forces in whatever order its worker threads finish, two runs with identical inputs and identical steps can disagree.
Scope the float claim

Float math is not "always nondeterministic." With an identical binary, identical flags, and the same architecture it can be perfectly reproducible. Divergence shows up across differing toolchains or hardware, or with fast-math. The mechanics live in IEEE-754 Floating Point; here it is enough to know fixed step alone does not buy you cross-machine determinism.

[Back to The Game Loop & Time](index.md)
