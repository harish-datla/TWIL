# Rigid-Body VAT

Shattering objects contain pieces that each move rigidly. Store one translation per piece per frame rather than one position per vertex, and store piece rotation as a quaternion. Each mesh vertex carries a piece ID and rest-pose pivot.

```text
vertexPosition = pieceTranslation + pieceRotation × (restPosition − restPivot)
```

Quaternions use four floats instead of a matrix’s 9–12, interpolate cleanly after normalization, and tolerate small filtering errors. Keep consecutive quaternions in the same hemisphere because `q` and `−q` represent the same rotation; a sign flip makes interpolation pass through zero. The piece count × frame count, rather than vertex count × frame count, controls texture size.
