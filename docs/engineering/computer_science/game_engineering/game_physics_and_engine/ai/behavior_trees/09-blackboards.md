# Blackboards

The Chase branch needs to know where the target is. The Attack branch needs the same target plus the ammo count. Perception found the target and the weapon system knows the ammo, and neither should have to know which nodes care.

Nodes in a behavior tree share that data through a blackboard: a key-value store, such as "target.distance" → 25, that any node can read from and write to. It holds the target entity, its last-known position, the agent's health, the ammo count, whether a perception query returned a result. Usually each agent gets its own blackboard instance, and some systems add shared parent scopes for squad-level data.

Unreal's blackboard is a first-class asset paired with the BT. You define typed keys (Object, Vector, Float, Bool and Enum, among others) in the blackboard definition. Decorators can observe specific keys and trigger re-evaluation when the value changes, which is the mechanism behind Unreal's event-driven BT model.[6]

Worked example

The widget's blackboard starts at target.visible = true, distance = 25, health = 80, ammo = 12. The root selector tries Retreat first: health < 30? is false at 80, so Retreat fails. Attack checks target.visible (true), then distance < 15 (false at 25), so Attack fails without checking ammo. Chase checks target.visible (true) and runs MoveTo, so the active behavior is Chase. Drop health to 29 and Retreat's first check passes, so it wins before Attack or Chase is looked at.

The widget below shows a behavior tree reading from a live blackboard. Adjust the sliders to change perception data; the tree re-evaluates on every change.
Live · Blackboard inspector
target.visible
visible: true
target.distance 25
health 80
ammo 12
active behavior
Chase
Drag the sliders and toggle visibility; the tree on the left re-evaluates on every change, colors each node by its result, and the readout names the active behavior. Retreat sits at the top and fires whenever health drops below 30, whatever else is true. Attack requires target.visible AND distance below 15 AND ammo above 0. Chase requires target.visible only. Patrol is the fallback. Set distance to 10 and the tree switches from Chase to Attack; set ammo to 0 and it falls back to Chase; turn visibility off and it falls through to Patrol. Then drop health below 30 from any of those states and everything else stops mattering, which is what putting a branch first in a selector actually means.
Check Your Understanding Q1 of 2

Two agents share a global blackboard. Agent A writes target.position when it spots an enemy. Agent B reads target.position when deciding where to move. Agent A dies. What happens to Agent B?
Agent B stops reading, because the key is cleared automatically when Agent A dies.
Agent B reads the stale position and moves toward a place that may mean nothing now.
Agent B crashes, because the target reference has become a dangling pointer.
0/2 correct
