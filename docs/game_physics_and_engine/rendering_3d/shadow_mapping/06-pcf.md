# PCF

A binary shadow lookup has a staircase edge. Percentage-closer filtering samples neighboring texels, performs the depth comparison at each tap, and averages the lit/shadowed results. Filter comparison results, not raw depths; averaging depths and comparing once only moves the hard edge.

A comparison sampler (`sampler2DShadow` with `compareEnable = VK_TRUE` and a compatible compare op) can bilinearly blend comparison results for one fetch. Larger kernels still need multiple taps. Keep sampler compare state and shader Dref usage consistent across GPUs.

A 3×3 kernel costs nine comparisons per fragment; at 1080p that is about 18.7 million comparisons per light. PCF softens edges but its penumbra has fixed width regardless of the distance between occluder and receiver. PCSS sizes the kernel from a blocker search for distance-varying softness.
