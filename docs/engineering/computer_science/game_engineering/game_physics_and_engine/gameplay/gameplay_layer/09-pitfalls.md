# Pitfalls
Deep inheritance for behavior
Cross-cutting concerns explode to 2^N classes. Compose instead.
"ECS is the only right model"
Three live models ship AAA. Unreal is composition + shallow inheritance.
Lapsed-listener leak
Unsubscribe on destroy, or the observer list keeps zombies alive.
"Event queue fixes reentrancy"
It hides it (stack unwinds). Spurious events still slosh around.
Deleting mid-iteration
Crash. Defer destruction to a frame boundary.
Raw pointers across frames
Dangle on reuse. Use {index, generation} handles.
Undefined update order
Readers see fresh state one tick and stale the next. Order by dependency, or double-buffer.
"The buffer removes the lag"
It makes the lag one tick per link in any order. Only dependency order removes it.
Spawns acting at once
An object added mid-loop updates in the tick it was born. Activate it at the end of the tick.
Copy-pasted prefabs
A fix to the base misses every copy. Inherit from a prototype and override only what differs.
Over-data-driving
Logic-as-data loses breakpoints and type checks. Keep a sane line.
Scripting the inner loop
Boundary overhead bites in hot per-entity logic. Keep it native.
State in chunk locals
A hot reload re-runs the chunk and resets them. Keep state in a table the reload reuses.
Unanchored coroutine
A Lua thread nobody references can be collected. Keep it in the registry while the host resumes it.
