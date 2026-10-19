# The Game-Object Model

Every enemy, bullet, and pickup needs storage. Options include plain entity structs in a vector, components attached to entities, and an archetype or struct-of-arrays ECS.

- Plain structs are simplest and work well for a few hundred entities.
- Components improve composition without requiring a full ECS.
- A small ECS improves cache locality and composition at larger scale, with more machinery.

Do not choose ECS reflexively. Three hundred 64-byte structs occupy only 19,200 bytes, while a movement pass over struct-of-arrays positions reads about 2,400 bytes. The layout difference matters increasingly at tens of thousands of entities; for a small arena game, a component struct or thin SoA is sufficient.
