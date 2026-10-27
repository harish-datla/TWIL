# The Memory Wall

VAT memory is:

```text
vertex_count × frame_count × bytes_per_vertex
```

A 30,000-vertex cloth at 30 fps for 10 seconds is 9 million texels, about 69 MiB in half-float RGBA. Texture dimensions also impose caps. A half-float 8192×8192 position texture is 512 MiB, so practical assets are much smaller.

Representative sizes: a 2,048-vertex, 64-frame pedestrian is about 1 MiB in half-float; BC6H can reduce that to roughly 128 KiB, but its 4×4 blocks span unrelated vertex/frame samples and can introduce large errors. Test compression per asset and consider vertex ordering. BC6H has no alpha, so packed normals cannot ride with positions.
