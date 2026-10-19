# Retrospective & pitfalls

What this isn't: production-complete. The capstone exercises every subsystem once. It skips open-world streaming and save/load, which the two Advanced modules after it cover (Open-World Streaming, Save Games); animation state machines and IK (Animation: Blending, State Machines & IK) and the audio DSP chain (Audio DSP), which earlier modules built but the slice doesn't need; and a full editor, GPU-driven rendering, LOD, and networking polish, which no module here covers.
Two accumulators
The loop owns the timestep. Physics/AI/anim run inside the one step.
Mutating sim in render
Render is read-only. Consume lerp/slerp; emit GPU commands only.
Sim calls the GPU
Breaks headless tests + interpolation. Sim makes a snapshot; renderer reads it.
Lerping rotations
Slerp or nlerp them; a raw lerp shrinks the rotation (the LBS failure).
Conflating the AI stack
Decide (BT) → plan (navmesh) → move (steering) → resolve (controller).
Deleting mid-iteration
Mark for death; reap at step end.
Blocking the audio callback
No locks/alloc/IO. Drain the SPSC ring wait-free.
"It's net-ready"
The shape is; bit-determinism isn't. Float results can vary across compilers and CPUs.
