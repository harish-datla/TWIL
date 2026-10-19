# Flags, Bitsets, and Bitboards


Bullets should hit players, enemies and walls, but fly through each other. Pickups should only react to the player. The most common bit work in a game answers questions like these with one integer per object and one bit per yes/no fact. Sixty-four of those facts fit in a single 64-bit word, which one load brings in and the §9 operators test and combine without a branch.
Collision layers and masks

Physics engines often decide whether two objects can collide with a pair of bitmasks. Each body has a category (one bit: which layer it's on) and a mask (the layers it accepts). Two bodies collide only if each one's category is in the other's mask: (catA & maskB) != 0 && (catB & maskA) != 0. Box2D's manual states the rule in exactly these terms, variable names and all, with one addition: a non-zero group index shared by both shapes overrides the masks[20]. Unity's layer collision matrix is a grid of checkboxes over the same kind of per-layer masks. Two ANDs replace a lookup table.

Worked example

Use the widget's six layers: Player is bit 0, Enemy bit 1, Terrain bit 2, Pickup bit 3, Bullet bit 4, Trigger bit 5.

A bullet's category is bit 4, 0b010000 (16). Its mask is every layer except Bullet: 0b101111 (47).

Two bullets: 0b010000 & 0b101111 = 0, so they pass through each other.

A bullet and a player who accepts everything: the player's category 0b000001 & 0b101111 = 1, and the bullet's category 0b010000 & 0b111111 = 16. Both non-zero, so they collide.
Live · Collision filter (the Box2D rule)
Body A
category (pick one)
Player
Enemy
Terrain
Pickup
Bullet
Trigger
mask (accepts)
Player
Enemy
Terrain
Pickup
Bullet
Trigger
cat = 000001 · mask = 111111
Body B
category (pick one)
Player
Enemy
Terrain
Pickup
Bullet
Trigger
mask (accepts)
Player
Enemy
Terrain
Pickup
Bullet
Trigger
cat = 000010 · mask = 111111
B.category & A.mask = 000010 & 111111 = 000010 → A accepts B
A.category & B.mask = 000001 & 111111 = 000001 → B accepts A
COLLIDE
Each body has a row of category chips (pick one) and a row of mask chips (toggle any), with both written out as bits; the two lines underneath are the two ANDs. It starts with a Player and an Enemy that both accept everything, so they collide. Set Body B to category Pickup accepting only Player: both directions still pass. Now make both bodies Bullets whose masks exclude Bullet: the ANDs come out 0 and they pass through each other, while a Bullet still hits a Player. The whole filter is two AND operations per pair.
Bitsets as component signatures

In an Entity Component System (ECS), an entity is just an ID and its data lives in separate component arrays. Many ECS designs give each entity a bitset "signature": bit k set means the entity has component k. A system that wants everything with Position and Velocity builds a query mask and accepts an entity when (signature & queryMask) == queryMask, meaning every required bit is present. Iterating the components an entity has means walking the set bits with the x & (x - 1) idiom from §9. The same pattern drives free-list allocators (a set bit is a free slot) and dirty-flag tracking.

Worked example

Position is bit 0, Velocity bit 1, Sprite bit 3. The query mask for "Position and Sprite" is 0b1001 (9).

An entity with Position, Velocity and Sprite has signature 0b1011. 0b1011 & 0b1001 = 0b1001, equal to the mask: match.

An entity with only Position and Velocity, 0b0011: 0b0011 & 0b1001 = 0b0001, which differs from the mask: no match.
Bitboards

Board games take this furthest. A bitboard packs an 8×8 board into a 64-bit word, one bit per square. In the common layout used here, squares are numbered row by row from a1 = 0 to h8 = 63. The payoff is that one shift moves every piece at once: shifting the whole board left by 8 slides every piece one rank north, and left by 1 slides it one file east. The catch is wraparound. Shifting east moves a piece on the h-file (the right edge) onto the a-file (the left edge) of the next rank, so you AND with a "not the a-file" mask to delete the wrapped bits[14]. A count of the set bits gives material; the index of the lowest set bit gives the next occupied square.

Worked example

A piece on e2 is square 12 (rank 1 × 8 + file 4). Shift left by 8: square 20, which is e3, one rank north.

A piece on h4 is square 31. Shift left by 1: square 32, which is a5, the wrong side of the board. Square 32 is in the a-file, whose bits notAFile (0xFEFEFEFEFEFEFEFE) has cleared, so the AND removes it: the piece walked off the edge.
Live · Bitboard mover (64-bit) pop: 5
north
south
east
west
clear

board
    0x0000000814081000
last move
    click a square, then shift

The board is one 64-bit value, a1 at the bottom left; the readout shows it in hex, and the header counts the pieces. Click squares to add or remove pieces, then press a direction to move all of them with one shift. Place a piece on the right edge (the h-file) and shift east: it vanishes instead of teleporting to the left edge, because notAFile masked away the wrapped bit, and the count drops by one. Drop the mask and that piece would reappear on the far side, one rank up.
Check Your Understanding Q1 of 1

An engine stores each entity component set as a 32-bit signature, one bit per component type. A system needs entities that have BOTH Transform (bit 0) and Sprite (bit 3). Which test selects them?
Build queryMask with bits 0 and 3 set, then accept when (signature AND queryMask) equals queryMask.
Accept when (signature AND queryMask) is non-zero.
Accept when signature equals queryMask.
0/1 correct

[Back to Bit Shifting from Scratch](index.md)
