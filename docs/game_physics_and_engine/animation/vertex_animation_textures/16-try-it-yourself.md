# Try It Yourself

Build a procedural cloth bake, then change precision and interpolation to see the tradeoffs:

```js
const vat = MPGVat.create({
  vertexCount: 24, frameCount: 64, bakeFramerate: 30,
  precision: '16-bit', interpolation: 'linear'
});
for (let frame = 0; frame < 64; ++frame)
  for (let v = 0; v < 24; ++v) {
    const u = v / 23, t = frame / 64;
    vat.set(v, frame, [u,
      Math.sin(u * 6 + t * Math.PI * 2) * 0.2 +
      Math.sin(t * Math.PI * 4) * u * 0.4, 0]);
  }
await vat.play({ speedX: 1.0, durationSeconds: 2 });
console.log(vat.stats());
```

Switch to 8-bit to see quantization error and point sampling to see stepped playback. Cubic interpolation costs more fetches and smooths velocity, but cannot recover motion absent from the bake.
