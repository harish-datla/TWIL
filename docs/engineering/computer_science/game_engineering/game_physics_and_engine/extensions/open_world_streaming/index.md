# Open-World Streaming

Open-world streaming keeps only the part of a large world near the player resident. This tutorial builds cell partitioning, prioritized loading, integration budgets, eviction and persistence, and far-field LOD on top of asynchronous file streaming and spatial partitioning.

Time: ~55 min  
Level: Senior  
Prerequisites: File Streaming, Spatial Partitioning, and Job Systems.  
Stack: C++ & Rust  
Track: Extensions beyond the capstone

## What you’ll be able to do

- Choose cell sizes and load radii for world density and player speed.
- Prioritize requests by distance, visibility, and predicted travel.
- Distinguish streaming hitches from pop-in and enforce an integration budget.
- Evict with hysteresis and persist world changes as deltas over cooked data.
- Keep distant geometry visible within a triangle budget using LOD and HLOD.

## Sections

- [The World Doesn’t Fit](01-the-world-doesnt-fit.md)
- [Partitioning into Cells](02-partitioning-into-cells.md)
- [Loading Radius & Priority](03-loading-radius-and-priority.md)
- [Getting the Bytes](04-getting-the-bytes.md)
- [The Integration Budget](05-the-integration-budget.md)
- [Eviction & Persistence](06-eviction-and-persistence.md)
- [LOD, HLOD & the Far Field](07-lod-hlod-and-the-far-field.md)
- [Pitfalls](08-pitfalls.md)
- [What’s Next](09-whats-next.md)
