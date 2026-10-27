# Omnidirectional Shadows

A point light shines in every direction, so a single 2D map cannot cover it. The standard solution is a depth cubemap with six 90° faces, sampled using the fragment-to-light direction. Many implementations store linear distance rather than projected depth:

```text
stored = length(fragment − light) / farPlane
test   = length(fragmentToLight) / farPlane
```

This scalar has the same meaning across all faces. A cubemap can cost up to six scene renders per point light, so engines limit shadow-casting point lights. Geometry-shader single-pass variants are not automatically faster because geometry-stage overhead can offset saved draws. A 1024²-per-face D32 cubemap uses about 24 MiB.
