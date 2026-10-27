# Making it a game

An enemy holds a pointer to its target, the target dies, and next step the enemy reads freed memory. A renderer and a physics engine aren't a game yet; the gameplay layer supplies the rest, including the fix for that crash. Entities are game objects built from components; the event bus decouples a hit from the health update, the UI, and the audio cue; generational handles keep an AI's reference to its target safe across that target's death (resolving a stale handle returns null instead of dangling); deferred destruction reaps the dead at step end. A game-state machine (menu / playing / paused / game-over) gates the sim[4], and waves spawn prefabs through the factory, configured by a Lua table so tuning needs no recompile.
A melee hit, end to end

An attack is one physics overlap query → an event on the bus → a damage-component update and an audio command pushed to the SPSC ring (Audio; the audio thread, fed lock-free, must never block) → and if health hits zero, the entity is marked for deferred destruction. Win when the survived-wave count reaches the target; lose when player health reaches zero. That one interaction touches physics, the event bus, audio, the component model, and, when it ends the last wave or the player, the state machine.

Worked example

In the arena below, a swing hits every enemy within 80 units for 30 damage, and enemies spawn with 30 health, so one swing kills everything in range; you can swing every 0.45 s. An enemy touching you drains 14 health a second, so your 100 health lasts about 7.1 s against one enemy and 1.4 s against five. The waves bring 5, 7, 9, 11 and 13 enemies, 45 in all, at 10 points each: a perfect run scores 450.
