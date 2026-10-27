# Diffuse and Energy Conservation

A surface cannot reflect more light than it receives. Lambert diffuse is `albedo/π`; specular reflection receives the Fresnel share, leaving diffuse:

```text
kD = (1 − F) × (1 − metallic)
```

The metallic factor removes diffuse from metals. Adding an unrestricted diffuse term and a full specular term makes materials glow and violates energy conservation.

The single-scatter microfacet model loses some energy at high roughness because light that bounces between facets is omitted. Multi-scatter compensation, such as Kulla–Conty methods or Filament’s approximation, addresses the darkening. The loss is a known model limitation, not necessarily an implementation bug.
