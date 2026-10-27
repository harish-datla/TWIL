# AO Is Not GI

A red wall beside a white floor should tint the floor pink when light bounces between them. Direct lighting plus AO leaves the floor white: ambient is merely scaled down, never transferred between surfaces.

Global illumination solves or approximates the recursive rendering equation:

```text
Lo = Le + ∫Ω fr × Li × (n·ωi) dωi
```

The incoming radiance `Li` is itself light leaving another surface. Direct lighting evaluates only source lights; GI approximates the remaining recursive indirect term. A flat ambient constant or single IBL probe is only a crude substitute. Surface albedo attenuates each bounce: a white room with albedo 0.8 retains roughly 80%, 64%, then 51% over three bounces, while albedo 0.3 retains 30%, then 9%.
