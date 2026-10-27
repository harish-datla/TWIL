# The Texture Layout

The vertex lookup coordinate must land at pixel centers. For `N` vertices:

```text
u = (vertex_id + 0.5) / vertex_count
```

For frame `j` in `F` rows:

```text
v = (time × bakeFramerate + 0.5) / frame_count
```

The half-texel offset is essential. Without it, bilinear filtering averages neighboring vertex columns. Along V, bilinear filtering can interpolate adjacent frames; along U, it must not interpolate adjacent vertices.

Disable mipmaps: adjacent vertex IDs have no spatial relationship and lower mips average unrelated vertices and frames. Use point sampling with manual frame interpolation, or bilinear sampling with U precisely at texel centers and V between frame centers.
