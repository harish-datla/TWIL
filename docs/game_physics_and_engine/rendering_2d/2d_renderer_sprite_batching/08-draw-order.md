# Draw Order

Opaque sprites can use the depth test and draw in any order. Transparent sprites cannot be represented by one depth value per pixel, so draw them after opaques and sort back-to-front, usually with depth testing enabled but depth writes disabled.

Sorting conflicts with batching: depth order may alternate textures. Atlases make the correct order cheap. For example, transparent sprites at depths 8 (texture A), 5 (B), and 3 (A) need order A, B, A—three batches. An atlas makes that order one texture and one batch. Intersecting or cyclic transparent geometry has no single correct order; order-independent transparency is an advanced approximation, not a default 2D solution.
