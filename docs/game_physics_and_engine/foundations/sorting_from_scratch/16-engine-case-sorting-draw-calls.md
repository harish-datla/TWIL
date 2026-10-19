# Engine Case: Sorting Draw Calls


Most renderers sort their draw calls before submitting them. State changes (binding a new shader, texture, or vertex buffer) cost CPU time in the driver and can stall the GPU, and the cost is per change, not per call. Five thousand draw calls in arbitrary order can mean thousands of shader changes; the same calls sorted by shader need only as many changes as there are distinct shaders. A per-frame sort is a cheap way to get that back.

Christer Ericson's 2008 pattern[1], still widely used, packs everything that matters into a single 64-bit sort key. One possible layout for opaque draws:
render target transparency bit shader id material id depth sequence number

Worked example

An opaque draw on render target 0 with shader 5, material 300, depth bucket 0x123456 and sequence number 7.

Shift each field to its place and OR them: (5 << 48) | (300 << 32) | (0x123456 << 8) | 7 = 0x0005012C12345607. Reading the hex left to right gives the fields in priority order: 0005 (target 0, opaque, shader 5), 012C (material 300), 123456 (depth), 07.

The bit order encodes the sort hierarchy: render target first, then the opaque/transparent split, then shader, then material, then a depth bucket. For opaque draws the depth field holds the depth bits directly, so draws sharing a shader and material come out front to back and early-Z rejects more hidden pixels. Transparent draws must be blended back to front across all of them, so for those the key uses a different layout: depth, bit-inverted so that far sorts first, moves above shader and material. Ericson describes exactly this swap of the depth and material fields. One sort then produces both orderings.

Once the key is packed, any integer sort works, and at thousands of keys a radix sort is a common choice. Unreal's core library includes RadixSort32, a stable radix sort on 32-bit keys[25], and its renderer packs mesh draw commands into 64-bit FMeshDrawCommandSortKey values[36]. Bevy tried swapping its comparison sorts for the radsort crate in 2022: the median sort_phase_system time on the many_cubes benchmark dropped from 0.728 ms to 0.094 ms, but the same change made it roughly ten times slower on two sprite benchmarks (0.184 to 1.95 ms on bevymark, 0.106 to 1.19 ms on many_sprites), a regression the issue left unexplained[2]. Measure on your own workload before switching.
Live · Pack a 64-bit draw-call key
primary sort field
shader binds avoided
7 / 16
material binds avoided
6 / 16
sort cost (ns/key)
~3
The boxes are sixteen draw calls in sorted order, colored by shader (hue) and material (lightness). Pick a key layout and watch them reorder; the readouts count shader and material binds avoided. The full layout avoids 7 of 16 shader binds and 6 of 16 material binds; the depth-first transparency layout avoids only 2 of each. Higher bits dominate the order. Render target first finishes one pass before the next starts; the transparency bit then separates opaque from transparent draws; shader next keeps pipeline changes down; depth last orders draws front to back within a shader and material. Transparent draws switch to the depth-first, back-to-front layout inside the same key. The stats compare shader and material changes with the worst case of one change per draw. The "sort cost" figure is the ~3 ns per key estimate from §12, not a measurement.

[Back to Sorting Algorithms](index.md)
