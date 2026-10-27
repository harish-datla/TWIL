# Instancing

Particles and tilemaps contain thousands of identical quads that differ only in position, scale, UV rectangle, and color. Upload the four quad corners once, bind per-sprite data at `VK_VERTEX_INPUT_RATE_INSTANCE`, and draw six indices with many instances. The vertex shader receives static corner attributes plus one instance record; `gl_InstanceIndex` can also index a storage buffer.

Instancing reduces bandwidth for uniform quads but does not remove texture-binding breaks. Dynamic vertices handle arbitrary geometry and interleaving more naturally; instancing is attractive for particles and tilemaps. For a record containing position (8 bytes), scale (8), UV rectangle (16), and color (4), 10,000 sprites use about 360 KB per frame versus 800 KB for four 20-byte vertices per sprite.
