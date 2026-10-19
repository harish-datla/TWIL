# Sprite VAT

Particle sprites are the simplest mode. Store one position per particle per frame. The static mesh contains a small card per particle; the vertex shader samples its particle column, places the card, and optionally billboards it toward the camera.

A 2,000-particle, 200-frame bake is 400,000 texels—about 3.1 MiB at half-float position storage. Sprite VAT gives one deterministic draw with no runtime particle simulation, but particles cannot react to gameplay or collide with geometry unless another system supplies that behavior.
