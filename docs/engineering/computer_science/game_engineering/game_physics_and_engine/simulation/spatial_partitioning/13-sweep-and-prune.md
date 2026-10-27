# Sweep and prune

Everything so far builds a structure over space and queries it. Broadphase collision detection asks a narrower question: which pairs of moving objects might be touching this frame? Sweep and prune answers it with no tree at all, by keeping the objects sorted.

Give every object an AABB and project it onto each axis, so each object becomes an interval on x, another on y, another on z. Two AABBs overlap exactly when their intervals overlap on every axis. Cohen, Lin, Manocha and Ponamgi's I-COLLIDE keeps three sorted lists, one per axis, each holding all 2n endpoint values, plus one overlap flag per pair per axis. A pair is a collision candidate when all three of its flags are set[22].

Sorting 2n values per axis per frame is O(n log n), which buys nothing over testing every pair at small n. The trick is that you never do that sort. At 60 Hz an object moves a fraction of its own size between frames, so last frame's list is already almost in order. That is temporal coherence, and insertion sort on an almost-sorted array costs time proportional to the number of inversions, not to n log n. The paper puts it in one line: "Insertion sort works well for previously sorted lists."[22]

The flags come along for free. A pair's overlap status on an axis can only change when the sort exchanges a maximum past a minimum: two minima swapping, or two maxima swapping, leaves every interval relationship intact. So the candidate set is maintained by the sort itself, in time proportional to the number of exchanges, and is never recomputed from scratch after the first frame.
Live · Sweep and prune under coherent motion and under teleports
coherent drift
teleport
Boxes: 12
pause
step
exchanges this frame
4
exchanges/frame avg
3.1
overlapping pairs
6
brute-force pair tests
66
Boxes drifting in the top panel, with a red line for every overlapping pair, and each axis's sorted endpoint list below. Compare the exchange readouts under coherent drift, then switch to teleport and change the box count. Both modes run identical code: one insertion sort per axis over the endpoint lists, with the overlapping-pair set updated only from the exchanges that sort performs and never recomputed. Right-pointing markers are interval minima, left-pointing are maxima, and anything the sort moved this frame is outlined amber. Under coherent drift at 12 boxes the lists arrive nearly in order and the sort averages about 3 exchanges a frame, against 66 brute-force pair tests. Switch to teleport and every box lands somewhere new: the lists arrive in essentially random order and the average settles near 255, just under the 276 that the expected inversion count of a random 24-element permutation predicts for two axes (each box's own minimum always precedes its maximum, which removes some inversions). Sweep and prune is then doing about four times the work of just testing every pair.

That last property is also the failure mode. The linear behavior is an expectation, not a bound: the paper states that in the worst case the number of exchanges on each axis is O(n²), "with an extremely small constant"[22]. Coherence is an assumption about the input, and games break it on purpose. A level streams in, a ragdoll comes apart, a teleport fires, a shockwave wakes a thousand sleeping bodies at once.

That is why PhysX ships a family rather than one algorithm. Its sweep-and-prune broadphase (eSAP) is documented as "a good generic choice with great performance when many objects are sleeping," whose "performance can degrade significantly though, when all objects are moving, or when large numbers of objects are added to or removed from the broad-phase." Multi box pruning (eMBP) "does not suffer from the same performance issues as eSAP when all objects are moving or when inserting large numbers of objects," but its "generic performance when many objects are sleeping might be inferior to eSAP, and it requires users to define world bounds." Automatic box pruning (eABP) manages those bounds itself, offering "the convenience of eSAP coupled to the performance of eMBP," and the documentation calls it "a good default choice for the broadphase"[14]. Each option bets on a different pattern of motion.
Check Your Understanding Q1 of 2

A sweep-and-prune broadphase finds that boxes A and B overlap on x and on y, but not on z. What does it report?
Nothing: a pair enters the candidate list only when all three axis flags are set at once.
The pair, since overlapping on two of three axes is close enough to be worth an exact test.
The pair, and lets the narrow phase decide, because the axis flags are only a heuristic.
0/2 correct
