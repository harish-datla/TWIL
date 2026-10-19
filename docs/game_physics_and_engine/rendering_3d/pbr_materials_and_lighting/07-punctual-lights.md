# Punctual Lights

Point, directional, and spot lights are punctual approximations: delta distributions that reduce the hemisphere integral to one BRDF evaluation per light. Point lights use inverse-square irradiance, directional lights provide constant irradiance, and spotlights add an angular cone.

Raw `1/d²` is physically appropriate but infinite at zero and extends forever. Engines use a windowed falloff. One useful range-limited form is:

```text
falloff = (1 − (d/range)^4)^2 / (d² + 1)
```

At range 10 and distance 5, this is about 0.034, close to `1/25`; at range it reaches zero and at distance zero it remains finite. Skip lights outside their range.
