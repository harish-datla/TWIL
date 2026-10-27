# Hot Reload


You nudge a material's roughness and want to see the change in the running game right away, without a restart and a level reload. The pipeline watches the source, re-cooks the changed asset and its dependents, and swaps the live copy without restarting. Bitsquid's tool sent a network message telling the engine to reload the changed file "nearly instantaneously"[3].
The two things that make hot reload hard

Dangling references: if anything holds a raw pointer into the old asset's memory, the swap invalidates it. This is exactly why generational handles matter; a stale handle is caught on next access instead of dereferencing freed memory. In-flight uses: an asset can be mid-render or mid-job when the reload fires, so a naive free-and-replace is a data race. Production systems typically refcount (count the holders and free at zero) or double-buffer (publish the new copy with an atomic pointer swap and keep the old one alive), and defer the free until no reader is active. And dependents must re-cook, not just the touched file: editing a shader include must reprocess every material that uses it.

Worked example

A texture is reloaded during frame 120, while two draw calls recorded earlier in that frame still reference the old copy, so its count is 2. The reload publishes the new texture for everything that asks from now on. When the GPU finishes frame 120, both draw calls release the old copy, the count reaches 0, and only then is its memory freed. Frame 121 draws with the new texture.

[Back to Asset Pipeline & Serialization](index.md)
