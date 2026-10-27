# The Two-Pass Idea

A point is shadowed when something lies between it and the light. Shadow mapping (Williams, 1978) turns that visibility question into two depth tests:

1. Render scene depth from the light into a depth texture.
2. Render from the camera, transform each fragment into light clip space, and compare its light-space depth against the map.

If the fragment is farther from the light than the recorded depth, a closer surface blocks it. The map is not an exact shadow mask: it is a finite-resolution, quantized raster of depth. Acne, aliasing, and peter-panning follow from that approximation.

For a crate over a floor, the map might store crate depth 0.40 for a texel. The floor fragment computes 0.62, so it is shadowed. Outside the crate, stored and computed depths both equal 0.62 and the floor stays lit.
