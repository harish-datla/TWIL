# Why inheritance failed

Every new kind of object used to mean a new class, and the tree of classes grew until nobody could place the next one. The historical model was a deep class tree: GameObject → Character → Player, GameObject → Item → Weapon. It breaks because behavior cuts across the tree. Where does a door that is also damageable go? An item that is also a light source? A flying enemy that needs AI, health, and a flight model? Single inheritance can't express "is-a" two ways, and multiple inheritance gives you the diamond and a layout nobody can whiteboard.
The combinatorial explosion, and what inheritance is
not

Try to express every combination by subclassing and N cross-cutting behaviors needs up to 2N leaf classes (FlyingDamageable, FlyingLight, FlyingDamageableLight…). Everything also drifts up into a fat base GameObject. But inheritance is not "always wrong": the failure is deep, behavior-bearing hierarchies for cross-cutting concerns. Shallow, purposeful inheritance ships AAA games; Unreal's ActorComponent → SceneComponent → PrimitiveComponent spine is deliberate (§3). (The ECS tutorial makes the cache-layout case against the OOP version; this is the architecture case.)

Worked example

With four cross-cutting behaviors (Damageable, Flying, LightSource, Networked), every non-empty combination needs its own class: 24 − 1 = 15. Add six more behaviors and it's 1,023 classes. As components it stays 10, one per behavior, attached in whatever combination an object needs.

The widget counts both as you toggle behaviors:
toggle behaviors:
Damageable
Flying
LightSource
Networked
Toggle behaviors on and watch the two counters: classes on the left, components on the right. With all four on they read 15 and 4. Left, the inheritance tree must add a class for every combination of enabled behaviors (the count races toward 2N). Right, the entity just adds one component chip and stays flat. The counters diverge, exponential vs linear, which is why composition won for cross-cutting behavior. (Shallow purposeful inheritance, like Unreal's component spine, is still fine.)
