# Loading a Mesh

Hand-authored cubes are useful for testing, but production geometry arrives from tools. glTF 2.0 is a runtime-ready, royalty-free interchange format. Its data model is:

```text
accessor → bufferView → buffer
```

A buffer is bytes, a buffer view is an offset/length/stride slice, and an accessor describes component type, element type, and count. Positions, normals, UVs, and indices are separate accessors. Handle unsigned byte, short, and int index types; respect interleaved `byteStride` and sparse substitutions.

glTF is right-handed, +Y up, +Z forward, uses meters, and declares counter-clockwise front faces. A cube with flat-shaded faces needs 24 vertices, not eight. A `VEC3` float position accessor has component type 5126; an unsigned-short index accessor has type 5123.

```rust
let (document, buffers, _images) = gltf::import("model.gltf")?;
for mesh in document.meshes() {
    for primitive in mesh.primitives() {
        let reader = primitive.reader(|b| Some(&buffers[b.index()]));
        let positions: Vec<[f32; 3]> = reader.read_positions().unwrap().collect();
        let normals: Vec<[f32; 3]> = reader.read_normals().unwrap().collect();
        let indices: Vec<u32> = reader.read_indices().unwrap().into_u32().collect();
        // Upload vertex and index data; handle optional attributes in real code.
    }
}
```

`cgltf` is a compact C loader; the Rust `gltf` crate supplies equivalent parsing.
