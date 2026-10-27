# Spatial Audio

Equal-power stereo panning maps an angle `θ ∈ [0, π/2]` to:

```text
L = cos θ,   R = sin θ,   L² + R² = 1
```

At center, both channels are 0.707 (−3 dB), while total power remains constant. Linear panning gives 0.5 per channel at center and a −3 dB total-power dip—the hole in the middle. Equal-power assumes the channels combine incoherently, as with two loudspeakers; a coherent mono fold-down can make its center +3 dB hot, which is why some mixers use compromise pan laws.

Distance attenuation should be clamped near the listener rather than using raw `1/r²`, which blows up at zero. A common inverse-distance model is:

```text
gain = reference / (reference + rolloff × (distance − reference))
```

With reference 1 m and rolloff 1, 10 m gives gain 0.1 (−20 dB), while anything closer than 1 m stays at full gain. Doppler changes resampling rate from relative velocity; HRTF convolves a source with head-related impulse responses for headphone 3D positioning.
