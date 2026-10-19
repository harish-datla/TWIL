# VAT vs Skeletal Animation

| Property | Skeletal | VAT |
| --- | --- | --- |
| CPU cost | Grows with bones, blends, IK, and characters | Near-zero per instance; GPU samples texture |
| Runtime flexibility | Blending, IK, ragdoll, interaction | Play/scrub baked clips only |
| Memory | Bone tracks scale with bones and clips | Vertex × frame product |
| Topology changes | No | Yes in fluid mode, at a memory cost |
| Best use | Heroes and interactive characters | Crowds, distant deterministic actors, baked sims |

Use skeletal animation for player-controlled or interactive characters and close-up/cutscene subjects. Use VAT for many distant, visually repeated actors. A practical engine supports both and switches representation by distance, as large crowd systems do.
