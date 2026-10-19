# Pitfalls
Blaming the disk for hitches
On an SSD the bytes are usually there already; the hitch is more often main-thread integration. Time-slice it.
Integrating a cell in one frame
One big activation drops a frame. Meter it to a per-frame budget.
Load radius = evict radius
Boundary thrash. Evict at a larger radius (hysteresis).
Predicting harder to beat pop-in
Stale velocity strands cells on turns. Keep the radius large; evict mispredicts cheaply.
One cell size everywhere
Too small churns; too large spikes. Tune to density, or go hierarchical.
Rewriting the cooked cell on change
Persist a delta over the read-only baseline; the save system serializes that.
"DirectStorage fixes stutter"
It speeds bytes-to-memory (pop-in), not the integration hitch.
Conflating pop-in with hitching
Throughput vs main-thread. Different causes, different fixes.
