# Interpolation

Other players' positions arrive 20 times a second, but your screen draws 60 or more frames a second. Draw each update as it arrives and they hop. The snapshot model doesn't predict other players: they stop, turn, and accelerate unpredictably, so every misprediction would pop (rollback in §9 and dead reckoning accept that cost). Instead, buffer their timestamped snapshots and render them at now − interpolationDelay, interpolating between the two snapshots that bracket that render time[8]. Source defaults to cl_interp 0.1 = 100 ms of view delay, derived from its 20 snapshot/s default and sized so a single lost snapshot still leaves two to interpolate between. The effective period is max(cl_interp, cl_interp_ratio / cl_updaterate), so with cl_interp set to 0, raising the update rate shrinks the delay well below 100 ms[9].
You see others in the past, deliberately

Interpolation delay is latency added on purpose, bought in exchange for smooth motion. It's interpolation between two real snapshots, not extrapolation, so it only coasts/extrapolates (and can overshoot) when snapshots are lost or arrive later than the buffer allows; Source caps that extrapolation at 0.25 s[9]. And it's coupled to lag compensation: the server subtracts this same delay when it rewinds time to validate your shots (§8).

Worked example

At Source's default 20 snapshots a second, snapshots are 50 ms apart, say stamped 0, 50, 100 and 150 ms (ignore travel time for the moment). With 100 ms of delay, at 175 ms the client draws the world as it was at 75 ms, halfway between the 50 and 100 snapshots. If the 100 ms snapshot is lost, the client blends between 50 and 150 instead, a 100 ms gap but still two real positions. Lose two in a row and it has nothing newer to blend toward, so it coasts.

A remote entity moving: compare raw snapshots (teleporting) to interpolation (smooth, but lagging), and drop packets to see it coast:
raw snapshots
interpolated
Packet loss 0%
Switch between raw and interpolated, then drag the packet-loss slider. Raw snapshots arrive ~10 times a second, so the entity teleports between them. Interpolated is smooth but visibly trails the true position (the faint ring) by the interpolation delay, 250 ms here, exaggerated for visibility. Raise packet loss and the interpolated entity coasts past gaps, the failure mode when too many snapshots drop.
