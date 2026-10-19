# ECS from Scratch

An Entity Component System separates identity, data, and behavior so gameplay code can iterate flat component storage and schedule independent systems in parallel. This tutorial builds sparse-set and archetype storage, query matching, structural changes, scheduling, and safe entity-ID recycling.

Time: ~40 min  
Level: Mid to senior  
Prerequisites: C++, Rust, cache and memory-model basics, and bitwise operations.  
Phase: 10 · Gameplay

## What you’ll be able to do

- Explain the frame-time cost of inheritance-based object models.
- Implement sparse-set and archetype ECS storage and choose between them.
- Defer structural changes with command buffers and match archetype queries.
- Schedule systems in parallel from declared read and write sets.
- Recycle entity IDs safely with generational indices.

## Sections

- [Why ECS](01-why-ecs.md)
- [A Short History](02-a-short-history.md)
- [The OOP Problem](03-the-oop-problem.md)
- [Entities, Components, Systems](04-entities-components-systems.md)
- [Storage Strategy 1: Sparse Sets](05-storage-strategy-1-sparse-sets.md)
- [Storage Strategy 2: Archetypes](06-storage-strategy-2-archetypes.md)
- [Iteration and Cache Coherence](07-iteration-and-cache-coherence.md)
- [Component Add/Remove](08-component-add-remove.md)
- [Queries and Query Caching](09-queries-and-query-caching.md)
- [Relationships and Entity References](10-relationships-and-entity-references.md)
- [Scheduling and Parallelism](11-scheduling-and-parallelism.md)
- [Generational Indices](12-generational-indices.md)
- [Case Studies](13-case-studies.md)
- [Pitfalls](14-pitfalls.md)
- [What’s Next](15-whats-next.md)
