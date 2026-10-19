# What AO Is and Is Not

Ambient occlusion approximates how much ambient light a point receives when nearby geometry blocks the hemisphere above it. It darkens crevices, contact points, and concavities, helping a crate read as resting on the floor.

```text
AO(p) = 1/π ∫Ω V(p,ω) × (n·ω) dω
```

`V` is 1 when a direction remains open within the chosen radius and 0 when blocked; cosine weighting gives overhead directions more influence. Open floor approaches AO=1, the base of a wall may approach 0.5, and a sealed corner may approach 0.25.

AO is not global illumination, because it never adds bounced light or color transfer. It is not a direct-light shadow; direct visibility belongs to shadow mapping. It is also a coarse approximation assuming uniform ambient input.
