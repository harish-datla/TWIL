# Fluid VAT

Fluid and smoke meshes can change topology each frame. Vertex 17 in one remeshed frame has no relationship to vertex 17 in the next, so the soft-body layout fails.

Fluid VAT ships a static pool of triangles sized for the busiest frame. Each frame maps pool vertices through a lookup texture to their positions in the position texture; unused triangles collapse out of sight. This costs two dependent fetches and memory for the worst frame plus lookup data. Dynamic UVs require per-frame UVs or generated mappings. Streaming a geometry cache can be larger and slower but is an alternative.
