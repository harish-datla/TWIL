# The Core Idea: A Texture Is a 2D Array

Treat the texture as numeric storage rather than an image. One pixel stores one vertex’s state at one frame. Columns are vertex IDs; rows are animation frames. The mesh topology stays static while vertex positions change in the shader.

For vertex 5 at frame 12 in a 16×32 texture, the texel is `(5,12)`. The vertex shader needs a vertex ID, commonly stored in a spare UV channel so it survives vertex reordering; `SV_VertexID` works when bake order is preserved. Time selects the frame row.

The same layout can store positions, normals, quaternions, or other per-vertex data. The rest of VAT is encoding, filtering, interpolation, and choosing a layout for topology and motion.
