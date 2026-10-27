# Normal Mapping

Normal maps fake small rivets and seams without adding geometry. They store normals in tangent space; a TBN matrix (tangent, bitangent, normal) transforms the sampled direction into world space. Encode `[-1,1]` into `[0,1]`, so a flat normal `(0,0,1)` is stored as `(0.5,0.5,1)`.

```glsl
vec3 sampledNormal = texture(normalMap, uv).xyz * 2.0 - 1.0;
// Flip sampledNormal.y here for a DirectX-style -Y map.
vec3 N = normalize(inNormalWS);
vec3 T = normalize(inTangentWS.xyz);
T = normalize(T - dot(T, N) * N);
vec3 B = cross(N, T) * inTangentWS.w;
vec3 normalWS = normalize(mat3(T, B, N) * sampledNormal);
```

Normal maps are directions, not colors: sample them through a linear/UNORM view, never sRGB. Also match the green-channel convention: OpenGL/glTF commonly uses +Y, DirectX-style maps commonly use −Y. Tangent handedness, stored in the tangent’s W, determines the sign of the bitangent. Dropping it breaks mirrored UV islands.
