# Case studies

Before writing your own engine, it pays to read how the shipping ones made the choices above.
1 · Box2D (Erin Catto)

Box2D[6] is the most widely read implementation of sequential-impulse dynamics. The v2.4 architecture is small enough to read end to end: a dynamic AABB tree for broad phase, GJK for distance queries, SAT-based contact generation for polygons, and the sequential-impulse solver with warm starting and accumulated clamping, run for a suggested 8 velocity and 3 position iterations per step[14]. Box2D's constraint formulation follows Catto's GDC presentations. Version 3 (2024) replaced the iteration counts with substepping and soft constraints. Shipped in 2D games such as Angry Birds, Limbo, and Crayon Physics Deluxe.
2 · Jolt Physics

Jolt Physics[9] is an open-source C++ rigid-body engine by Jorrit Rouwe (v1.0 in 2022). Its broad phase is a quadtree of AABBs, split into configurable layers, that bodies can update from several threads without locks. Narrow phase uses GJK and EPA. The solver is a SIMD-optimized sequential-impulse solver with warm starting (10 velocity and 2 position iterations by default), run per simulation island as jobs, with large islands split into groups that can be solved in parallel. It offers opt-in cross-platform determinism at roughly an 8% cost. Its README lists Horizon Forbidden West and Death Stranding 2 among the games that use it.
3 · PhysX: PGS and TGS

NVIDIA PhysX (open source) offers two iterative solvers, projected Gauss-Seidel (PGS) and TGS, both available on CPU and GPU[12]. PGS runs all its iterations on the same constraints and then integrates; TGS substeps, integrating after every position iteration, and the documentation calls it the generally recommended solver. The two also differ in friction: PGS clamps the two tangent directions separately, while TGS clamps their combined impulse. Gyroscopic forces are off unless a per-body flag enables them.
4 · Havok

Havok, now part of Microsoft, is a long-running commercial physics middleware whose solver internals are not publicly documented. Its site lists titles such as Assassin's Creed Odyssey, The Legend of Zelda: Breath of the Wild, and The Elder Scrolls IV: Oblivion Remastered among the games built on Havok Physics[15].
