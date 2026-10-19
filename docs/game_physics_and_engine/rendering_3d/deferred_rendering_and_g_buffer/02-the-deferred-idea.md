# The Deferred Idea

Deferred shading splits rendering into two passes. The geometry pass rasterizes the scene and writes each visible surface’s material data, normal, and depth into multiple render targets—the G-buffer. The lighting pass runs once per screen pixel, reads those targets, and evaluates every light for the surface that survived the depth test.

With three overlapping spheres and 50 lights, forward may evaluate 150 light contributions even though two spheres are hidden. Deferred still runs its cheap G-buffer shader for overdrawn fragments, but evaluates the 50 lights once for the visible sphere. The win is removing redundant lighting, not removing geometry rasterization.

The cost is writing and then reading every G-buffer target. Deferred is a bandwidth-for-shading trade, not an unconditional speedup.
