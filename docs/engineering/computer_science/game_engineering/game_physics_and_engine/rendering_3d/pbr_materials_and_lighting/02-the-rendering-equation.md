# The Rendering Equation

Kajiya’s rendering equation describes outgoing radiance:

```text
Lo = Le + ∫Ω fr(ωi, ωo) × Li(ωi) × (n · ωi) dωi
```

`Lo` is light leaving toward the viewer, `Le` is emission, `fr` is the BRDF, `Li` is incoming radiance, and `n·ωi` is the cosine that accounts for angled illumination. The integral sums all directions in the hemisphere above the surface.

Real-time rendering cannot integrate every direction at every pixel. It approximates the integral with a finite set of punctual lights and image-based lighting. Keep the cosine outside the BRDF: `Lo += BRDF * radiance * NdotL`.

For a matte white surface with albedo 0.8, light radiance 2.0, and a 60° incidence angle, Lambert is `0.8/π = 0.255`; multiplying by radiance and `cos(60°)=0.5` gives 0.255 outgoing contribution.
