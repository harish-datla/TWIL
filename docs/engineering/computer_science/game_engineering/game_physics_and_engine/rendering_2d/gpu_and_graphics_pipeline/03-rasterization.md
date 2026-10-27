# Rasterization

Rasterization decides which sample points a triangle covers. Pineda’s edge function makes the decision cheap. For edge `a → b` and point `p`:

```text
E(p) = (p.x − a.x) × (b.y − a.y)
     − (p.y − a.y) × (b.x − a.x)
```

The value is twice the signed area of the triangle `(a, b, p)`. Test all three edges; a sample is inside when all values have the same sign as the triangle’s area. Divide the three edge values by the doubled area to obtain barycentric coordinates. They sum to one and interpolate vertex attributes such as colors and UVs.

For `v0=(0,0)`, `v1=(0,4)`, `v2=(4,0)`, the doubled area is 16. Pixel center `(1.5,0.5)` has edge values 8, 2, and 6, producing barycentrics `0.5`, `0.125`, and `0.375`. A point such as `(3.5,2.5)` fails one edge and is outside.

The top-left rule resolves samples exactly on shared edges, so adjacent triangles draw a shared edge once rather than leaving gaps or double-shading. “Inside” means the sample point, normally the pixel center, passes the test; it does not mean the triangle merely overlaps the pixel.
