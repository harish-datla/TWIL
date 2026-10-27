# Update order

A turret aims at a moving target, and whether it aims at where the target is or where it was depends on which of them updated first. Gameplay ticks inside the fixed-step block of the game loop (input, AI, gameplay, physics run at the constant dt; rendering interpolates with alpha outside it). The Update Method pattern gives each object an update(dt) called once per tick[7].

The loop calls those methods one at a time, so a tick is a sequence of turns. If A comes before B in the list, A sees B's state from the previous tick and B sees A's new state[7]. Neither object has a bug; the order decides what each one reads. It shows up whenever one object reads another's state during its own update: a turret reading its target, a trailer following a truck, a camera following the player.

Worked example

A target runs at 10 m/s with a 60 Hz tick. If the turret updates before the target, it aims at last tick's position, 10 ÷ 60 = 0.17 m behind where the target ends the tick. Fix the order so the target always moves first, or have the turret lead its shot.

A chain of dependent objects makes it worse, because the lag adds up link by link. The widget is a convoy: a lead truck and five trailers, each placed 3 m behind the one ahead at the moment it updates, at 60 ticks a second.

Worked example

At 10 m/s the truck moves 10 ÷ 60 = 0.17 m per tick. Update the convoy front to back and each trailer reads a position that has already moved this tick, so each sits exactly 3 m behind the one ahead. Update it back to front and each trailer reads the one ahead before that one has moved, one tick out of date: 0.17 m per link, and the fifth trailer ends up 5 × 0.17 = 0.83 m behind its proper spot.
leader first
reverse
shuffled
read last tick
speed 10 m/s
order: leader firstlag per link (ticks): 0 · 0 · 0 · 0 · 0last trailer behind: 0.00 m (0 ticks)
The convoy drives right; the number above each vehicle is its turn in this tick's update order, and the number below each trailer is how far it sits behind its zero-lag spot (the dashed outline). Switch between the four orders, then drag the speed. Leader first reads 0 ticks of lag on every link. Reverse reads 1 per link, and at 10 m/s the last trailer is 0.83 m behind. Shuffled picks a new order every tick, so each link's lag jumps between 0 and several ticks (a trailer that reads a fresh position can catch up a whole backlog at once) and the trailers jitter. Read last tick updates front to back but has every trailer read the previous tick's copy, so it shows the same 1-per-link lag as reverse, now independent of the order. Lag in metres grows with speed: at 30 m/s the last trailer is 2.5 m back, almost a whole gap.

There are three fixes, and engines use all of them:

    Order by dependency. Update whatever is read before whatever reads it. Unreal sorts ticks into groups that run in a fixed order (pre-physics, during physics, post-physics, post-update work), finishes each group before starting the next, and lets one actor or component declare that it must tick after another with a tick prerequisite[8]. Unity's Script Execution Order setting runs script classes from the lowest number to the highest[9]; it ranks classes, so it can't put trailer 2 before trailer 3 when both run the same script.
    Read last tick's state. Keep a copy of each object's state from the previous tick and have every reader use the copy, the Double Buffer pattern. Order stops mattering because everyone reads the same snapshot, but every link now lags exactly one tick. In Nystrom's version, "an actor will only see a slap in the frame after it was actually slapped"[10]. In the convoy that's the same 0.83 m as back to front, in any order.
    Accept the lag where nobody can see it. Unreal's own example puts a HUD meter in the during-physics group and notes the HUD will be one frame behind the game[8]. For a meter, nobody notices.

An undefined order can change every tick

If nothing defines the order, nothing stops it changing between ticks as objects are added, removed or moved between slots. Then a link is 0 ticks behind on one tick and 2 on the next, which is the shuffled case: a fixed wrong order lags smoothly, an unspecified one jitters. Two more rules come with update methods. The code must store its own cross-frame state, because returning from update() loses the execution position[7] (§8 shows how coroutines keep it). And don't modify the object list while iterating it, which is the lifetime problem in §6.
