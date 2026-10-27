# Cluster Streaming: Doing for Geometry What SVT Did for Textures


A scanned statue arrives from the art team with millions of triangles, and the level places hundreds of them. Most are far away, where a few hundred triangles would look identical, but the one the player walks up to needs every detail. Textures had been virtualized for over a decade before geometry caught up. The best-known system is Unreal Engine 5's Nanite[9], demoed in 2020 and shipped in 2022. The structure parallels SVT, but the unit is a triangle cluster, not a texel tile.
The shape of the system

    Clusters of ~128 triangles. A mesh is preprocessed into clusters: small connected groups of triangles with a shared bounding sphere, one sphere enclosing them all for cheap visibility tests (many cluster pipelines also store a normal cone, the spread of directions the triangles face, for back-face culling). The cluster is the unit of culling and LOD selection (picking a simpler version of the mesh when it's far away); streaming moves pages of clusters (below).
    A DAG of LOD groups. A DAG (directed acyclic graph) is a graph whose arrows never loop back, here from each coarse group to the finer groups it can be refined into. Adjacent clusters are merged and simplified to produce a coarser LOD. This is recursive: clusters → groups of clusters → groups of groups, all the way up to a single root cluster representing the whole mesh.
    Pages of 128 KB. Cluster groups within the same LOD that share spatial locality are packed into 128 KB pages. The page is the unit of streaming I/O. Karis et al.'s SIGGRAPH talk[9] describes the compression scheme and the page layout in detail.
    GPU-driven cluster culling. Every frame, compute passes walk a hierarchy over the cluster groups, pick the clusters whose simplification error is small enough on screen, and emit the list to rasterize. Clusters that aren't resident are replaced with their nearest resident ancestor, so the surface is always rendered, just at coarser detail.

root (LOD 2) LOD 1 group A LOD 1 group B leaf 0 leaf 1 leaf 2 leaf 3 leaf 4 leaf 5 coarse fine always resident streamed

Worked example

A 1,000,000-triangle statue splits into 1,000,000 ÷ 128 ≈ 7,800 leaf clusters. In Nanite each step up the hierarchy roughly halves the triangle count[9], so about 13 halvings reach a single root cluster, and the whole hierarchy holds about twice the leaf count, around 15,600 clusters. Across a courtyard, the root's 128 triangles can stand in for the full million. Walk up to the statue's face and the culling pass asks for the leaves covering that part only; the back and the base stay coarse.

The hierarchy is what makes streaming work without visible pop. A camera far from the model only ever asks for the root and the coarse intermediate levels, which are always resident. As the camera moves closer, the GPU's cluster-culling pass starts asking for fine-grained leaves; the streamer loads the corresponding pages; the renderer falls back to the coarser ancestor for any leaves that aren't yet in. Pop-in becomes a gradual sharpening rather than a hard appearance.
Live · Cluster Streaming
camera distance 10.0
streaming bandwidth (pages per 100 ms) 2
Dolly ▶
Pause ⏸
Reset ⟲
visible clusters
64
resident pages
5
LOD fallbacks
64
The surface is drawn as cluster cells at the detail the camera distance calls for. Press Dolly, then lower the bandwidth to 1 and look for amber outlines. Dolly sweeps the camera in and back out. Each cluster's page streams separately; until it arrives, the renderer draws the nearest resident ancestor, so missing detail shows up as a bigger, coarser block with an amber outline. "LOD fallbacks" is the number of clusters drawn that way right now, which tells you whether streaming bandwidth is keeping up with how fast the camera changes distance. Pages finer than the camera needs are freed as it pulls back, so the next approach streams them in again.
The same pattern elsewhere

The recurring pattern: a fixed-budget physical cache, a logical-to-physical indirection, and a feedback signal that decides what to fill it with. SVT applies it to texels streamed from disk; Nanite applies it to triangle clusters. Unreal's virtual shadow maps apply it to data generated on the GPU instead: a 16k × 16k virtual shadow map split into 128 × 128 pages, with only the pages that on-screen pixels need allocated and rendered, and cached between frames[47].

[Back to File Streaming for Game Engines](index.md)
