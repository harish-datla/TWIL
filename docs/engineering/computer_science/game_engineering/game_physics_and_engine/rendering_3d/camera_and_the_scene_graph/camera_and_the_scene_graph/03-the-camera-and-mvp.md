# The Camera and MVP

The view matrix is the inverse of the camera’s world transform. `lookAt(eye, center, up)` constructs it directly. Common camera models are orbit (spherical coordinates around a target) and free-fly (position plus yaw/pitch).

An orbit camera 5 m from the origin at 45° yaw and 30° pitch has approximately `eye=(3.06,2.5,3.06)`. `lookAt` turns that into a view matrix.

For column vectors, the transform is:

```text
clip = P · V · M · position
```

Model places the vertex in the world, view changes to camera space, and projection applies perspective. `P` and `V` are shared by every object in a frame; `M` changes per object. Row-vector libraries write the equivalent chain in the opposite notation.
