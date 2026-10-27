# From 2D to 3D

The instance, device, swapchain, and synchronization remain unchanged. Four additions turn the sprite renderer into a 3D renderer:

- Orthographic projection becomes perspective, producing a non-unit `w` and foreshortening after the divide.
- A depth buffer becomes mandatory because arbitrary 3D overlap cannot be solved by draw order.
- Vertices gain a Z coordinate and a normal for later lighting.
- Each object needs a model matrix, which raises the question of how to feed many matrices to the GPU.

A 3D vertex with position, normal, and UV is 32 bytes. A flat-shaded cube uses 24 vertices (not eight, because each face needs its own normal), 36 16-bit indices, and a 32-bit depth buffer adds about 8.3 MB at 1920×1080.
