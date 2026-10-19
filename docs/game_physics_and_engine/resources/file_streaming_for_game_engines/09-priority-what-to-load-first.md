# Priority: What to Load First


The player sprints around a corner, and the wall in front of them is still a blurry placeholder because the streamer spent its last second loading tiles behind them. Eviction is the question of what to remove from RAM. Priority is the question of what to load first. They're symmetric: the streamer is constantly choosing between candidate reads, and the order matters as much as the reads themselves.

The priority of a pending tile is a function of player state. Typical inputs:

    Distance to player. Closer is more urgent.
    Screen-space size. A faraway tile that covers a lot of the screen (because the camera is looking at it) outranks a closer tile behind the player.
    View frustum bias. The frustum is the region of space the camera can see. Tiles in the camera's cone get a multiplier; tiles outside it get a discount.
    Velocity prediction. If the player is moving in a direction, weight tiles in that direction higher, since that's where the player will be by the time the loads land.
    Importance flags. Quest-critical assets and player-character pieces outrank scenery.

A priority score folds those inputs into one number per pending tile, so the streamer can sort candidates and load the highest first. A typical form multiplies them:
eq. 3 · a typical priority score score = importance × max(0, 1 − d) × (1 + frustum_bonus ) × (1 + velocity_bonus )

In words: start from how important the asset class is, scale it down with distance (to zero at the edge of the load range, since d is distance divided by that range), then boost it if the tile is in the camera's view and again if it lies in the direction the player is moving. The exact form varies by engine, but the spirit is the same: a few cheap geometric heuristics fold into one number, and the streamer pops candidates off a max-heap (a priority queue that always hands back the largest) by that number. There's no "right" formula. You tune it against the worst-case traversal in your game and look for where pop-in appears.

Worked example

The widget's settings: a load range of 6.3 tiles, a view-cone bonus of 1.5 and a velocity bonus of 1.0, with importance 1. A tile 3 tiles straight ahead has d = 3 ÷ 6.3 = 0.48, so it scores (1 − 0.48) × (1 + 1.5) × (1 + 1.0) = 2.6. A tile 2 tiles behind the player has d = 0.32 but gets neither bonus: 1 − 0.32 = 0.68. The farther tile ahead loads first, by almost four to one.
A streamer working through the heap

The player (yellow dot) walks through a grid of tiles. Each missing tile inside the dashed load range has a priority score from the equation above, shown as brightness; every 100 ms the streamer starts loading the highest-scoring tiles its budget allows. Drag the player with the pointer (or focus the canvas and use the arrow keys); the view cone follows the direction of movement and the priorities re-rank live:
Live · Priority Streaming
I/O budget (tiles per 100 ms) 2
view cone bonus 1.5×
velocity bonus 1.0×
Walk ▶
Pause ⏸
Reset ⟲
resident
120
pending
0
pop-in events
0
wasted loads
0
Green tiles are resident, amber ones are loading, and dark ones are missing, brighter for a higher score; the shaded wedge is the view cone. Press Walk, then try budgets of 1 and 3. Each load takes one second, and the player walks two tiles a second. "Pop-in events" counts tiles that finish loading while inside the view cone (or right next to the player), where the player would see them appear late. At the default budget the streamer only just keeps up: drop it to 1 and pop-in climbs steeply, raise it to 3 and it nearly disappears. The two bonuses matter at that margin, where turning them up moves the tiles ahead of the player to the front of the queue and typically cuts pop-in by a third or more over a minute of walking (single runs vary with the random path). "Wasted loads" counts tiles that land outside the dashed load range because the player moved or turned away while they were in flight: those are the lone green tiles that sometimes appear far behind the player, and they stay until they fall past the eviction range. Production streamers cancel requests that drop out of range; the demo deliberately doesn't, so the failure mode stays visible.
Hysteresis matters

Hysteresis is the trick of using different thresholds for entering and leaving a state. Without it, a tile right on the eviction-threshold boundary can be loaded, evicted, loaded, evicted as the player wiggles. The fix is to keep an asset resident until it falls well below the load threshold, typically at 1.5x or 2x the distance (the widget above uses 1.3x). The same pattern applies to requests: don't request a tile until it crosses a stricter "I'm about to need this" threshold, not the looser "this might be visible" one.

[Back to File Streaming for Game Engines](index.md)
