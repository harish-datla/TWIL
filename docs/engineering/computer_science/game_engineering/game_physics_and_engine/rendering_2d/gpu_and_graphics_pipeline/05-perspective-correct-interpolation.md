# Perspective-Correct Interpolation

Perspective projection keeps straight lines straight but does not preserve distances. A value linear on a 3D surface is not generally linear in screen space, so interpolating UVs directly with screen-space barycentrics makes a textured quad kink along its diagonal.

Suppose an edge goes from `(w=1,u=0)` to `(w=3,u=1)`. At the screen midpoint, plain interpolation gives `u=0.5`. Correct interpolation blends `u/w` and `1/w`: halfway gives `u/w=1/6` and `1/w=2/3`, so `u=(1/6)/(2/3)=0.25`.

```text
u = (Σ λᵢ × uᵢ / wᵢ) / (Σ λᵢ / wᵢ)
```

## Software rasterizer

```cpp
struct Vertex {
    float screenX, screenY;
    float oneOverW;
    float uOverW, vOverW;
};

static float edge(const Vertex& a, const Vertex& b, float px, float py) {
    return (px - a.screenX) * (b.screenY - a.screenY)
         - (py - a.screenY) * (b.screenX - a.screenX);
}

void rasterize(Vertex v0, Vertex v1, Vertex v2,
               uint32_t* framebuffer, int width, int height) {
    float area = edge(v0, v1, v2.screenX, v2.screenY);
    if (area <= 0) return;
    float invArea = 1.0f / area;
    for (int y = 0; y < height; ++y)
    for (int x = 0; x < width; ++x) {
        float sx = x + 0.5f, sy = y + 0.5f;
        float w0 = edge(v1, v2, sx, sy);
        float w1 = edge(v2, v0, sx, sy);
        float w2 = edge(v0, v1, sx, sy);
        if (w0 < 0 || w1 < 0 || w2 < 0) continue;
        float l0=w0*invArea, l1=w1*invArea, l2=w2*invArea;
        float invW = l0*v0.oneOverW + l1*v1.oneOverW + l2*v2.oneOverW;
        float uW = l0*v0.uOverW + l1*v1.uOverW + l2*v2.uOverW;
        float vW = l0*v0.vOverW + l1*v1.vOverW + l2*v2.vOverW;
        framebuffer[y * width + x] = shade(uW / invW, vW / invW);
    }
}
```

This teaching rasterizer omits depth, 2×2 derivatives, near-plane clipping, fixed-point tie-breaking, MSAA, SIMT, and tiling. NDC depth is the important special case: it is already divided by `w` and is screen-linear, so depth itself is interpolated linearly.
