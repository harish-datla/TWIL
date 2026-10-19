# Normals: Three Approaches

Deforming positions without deforming normals leaves lighting frozen in the rest pose. Common choices are:

1. **Pack normals in position alpha:** Octahedrally encode a unit normal into two values. One texture and one sample, but medium quality, especially on glossy surfaces.
2. **Separate normal texture:** Store `normal*0.5+0.5` with the same vertex×frame layout. More memory and a second sample, but simple and generally adequate.
3. **Rotation texture:** Store a per-frame quaternion that carries normal and tangent orientation. This supports tangent-space normal maps whose tangent frame deforms with the mesh.

Object-space normal maps are a shortcut: rotate them using the animated normal’s rest-to-current rotation, accepting an approximation around the normal axis. Choose based on distance, glossiness, memory, and whether tangent-space detail is needed.
