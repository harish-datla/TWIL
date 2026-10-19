# Lockstep & determinism

In deterministic lockstep, every peer runs the identical simulation and only inputs (commands) cross the wire, so bandwidth is proportional to input size, not world-object count[4]. Age of Empires passed commands rather than per-unit state to hit its 1,500-unit target on a 28.8k modem; passing per-unit position and status would have capped it at about 250 moving units[5]. Commands are scheduled a couple of turns ahead so transmission overlaps simulation.

Worked example

A 28.8 kbit/s modem moves 28,800 / 8 = 3,600 bytes a second. Say each unit's position and status take 8 bytes: 1,500 units are 12,000 bytes, more than 3 seconds of modem time for a single update. A command such as "these 40 units attack that building" is a few dozen bytes, whatever the units then do for the next minute.
Two hard costs: the slowest peer, and determinism

Lockstep runs at the speed of the slowest, laggiest peer (everyone needs turn N's inputs before simulating turn N), and a single non-deterministic divergence desyncs everyone, permanently, and compounds over time[1]. Across machines, the classic culprit is floating point: transcendentals not correctly rounded, non-associativity under compiler/SIMD reordering, -ffast-math, x87 vs SSE in legacy 32-bit builds, and FMA contraction and libm differences across x86/ARM (the Floating Point tutorial covers the mechanism)[6]. The fixes are fixed-point math or tightly controlled float. Floats aren't the only source: Age of Empires' programmers had to keep the number of calls to the (seeded, synchronized) random generator identical on every machine[5], and uninitialized memory or iteration over pointer-keyed containers diverge the same way. A fixed timestep is necessary but not sufficient, and using doubles doesn't fix it. Those two costs keep many engines off lockstep.
