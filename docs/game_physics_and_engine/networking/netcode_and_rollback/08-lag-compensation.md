# Lag compensation

You line up a headshot on a player running past a doorway and click. Your screen showed them in the doorway, but by the time your shot reaches the server they're behind the wall. Should it hit? When validating a hit, the authoritative server rewinds the other players to where the shooter saw them. To do that it needs the moment the shooter was looking at, and Valve estimates it as:

Command Execution Time = Current Server Time − Packet Latency − Client View Interpolation

In words: start from the server's clock now, go back by the time the shot's packet took to arrive, then go back again by the interpolation delay the shooter was seeing other players with.

The formula subtracts the interpolation delay from §6: the two systems are coupled[9]. The server keeps about 1 second of position history, moves the candidates back, tests the hit, and restores them.

Worked example

The shot arrives at server time 10.000 s. It took 40 ms to get there, and the shooter renders other players 100 ms in the past, so the server rewinds everyone to 10.000 − 0.040 − 0.100 = 9.860 s. The target stepped behind the wall at 9.900 s, but at 9.860 s they were still in the doorway, so the hit counts, even though on the server the target had been behind the wall for 100 ms.
It favors the shooter, by design

Lag compensation makes you hit what your screen showed, at the cost of the target: you can be killed after you have already ducked behind cover, because on the shooter's machine you were still exposed. Valve is explicit that this "can't be solved in general because of the relatively slow packet speeds." The trade is shooter feel against target fairness. Source rewinds only players; every other entity stays at its current position[9].
