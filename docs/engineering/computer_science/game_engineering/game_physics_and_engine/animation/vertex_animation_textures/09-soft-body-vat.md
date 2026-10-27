# Soft-Body VAT

Soft-body VAT assumes stable topology: vertex IDs persist across frames while positions and optional normals change. Bake the simulation, export a static mesh with normalized vertex ID in a spare UV channel, and sample/decode in the vertex shader.

Cloth, banners, jiggle, deforming foliage, and baked character cycles fit this mode. A 192-vertex cloth with 32 half-float frames uses only `192×32×8 = 49,152` bytes, shared by every instance. Runtime cost is one texture sample and decode per vertex, but every instance repeats the same baked motion unless given a time offset.
