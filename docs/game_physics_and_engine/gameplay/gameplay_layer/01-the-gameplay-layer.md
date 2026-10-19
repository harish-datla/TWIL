# The gameplay layer

Every game object question (what is a door, how does it hear a switch, when does it update, who decides it exists) lands in one layer of the engine. Gregory calls this the gameplay foundation layer: the game object model, the world structure, the event system, and the scripting system, all sitting on top of the low-level engine[1]. It's distinct from ECS-as-storage: ECS is how component data is laid out and iterated; the gameplay object model is how you think about and wire up a thing in the world. An ECS is one way to implement the object model, not the only one.

Worked example

The designer's shootable door, built from parts: a Transform (where it is), a Mesh (what it looks like), a Collider (what it blocks), a Door (open and closed states), and a Health (how many hits it takes). Five components, none of them new code; the only new thing is the combination.
The four questions this module answers

How is a game object modeled (§2–3)? How do its parts communicate without hard coupling (§4)? When and in what order does it update, and how does it live and die safely (§5–6)? And how is its definition and behavior moved out of compiled code into data and script (§7–8)? The loop itself (accumulator, fixed dt, alpha interpolation) and the storage layout (ECS) are prerequisites, not re-covered here.
