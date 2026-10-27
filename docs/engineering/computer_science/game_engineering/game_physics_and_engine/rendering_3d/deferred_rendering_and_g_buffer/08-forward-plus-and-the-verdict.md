# Forward+ and the Verdict

Forward+ keeps forward shading’s hardware MSAA, transparency, and per-material flexibility while borrowing tiled or clustered light lists. Its usual structure is depth pre-pass, compute light culling, then forward shading where each fragment loops only its tile’s lights.

Forward+ may draw geometry twice, but the depth pre-pass writes only depth—about 8.3 MB at 1080p—rather than a 33.2 MB 16-byte G-buffer plus a readback. Both approaches can use the same culling data.

No architecture is universally best:

- Few lights: forward or Forward+ avoids G-buffer bandwidth.
- Many dynamic lights: tiled or clustered culling helps, whether deferred or forward.
- Heavy transparency or MSAA: Forward+ is often attractive.
- Many opaque surfaces and expensive lighting: deferred can amortize its fixed targets.

Hybrid renderers commonly use deferred opaque lighting and a forward transparent pass.
