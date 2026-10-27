# Local avoidance

Two guards walk down the same corridor in opposite directions. Each one's path is fine; together they're about to walk through each other, and neither path knew the other existed. A velocity obstacle (VO) is the set of an agent's velocities that, relative to a neighbor, lead to a collision: a cone in velocity space with its apex at the neighbor's velocity, cut off at a time horizon in most game implementations. Pick a velocity outside it and you avoid the collision, provided the neighbor keeps its velocity. The concept comes from Fiorini and Shiller's robotics work[6].
Plain VO oscillates; reciprocity fixes the two-agent case; ORCA is the LP formulation

If two agents each steer fully out of the other's VO, both dodge; on the next update each sees that the other has already moved aside, finds its preferred velocity clear again, and snaps back; then both are on a collision course and dodge again. That is VO oscillation. (It is different from a reciprocal dance, where the agents disagree about which side to pass on.) RVO (van den Berg, Lin, and Manocha) has each agent take half the responsibility: its new velocity is the average of its current velocity and one outside the VO. For two agents that each pick the closest such velocity, the paper proves the motion is collision-free and oscillation-free, and both pick the same side[6]. With three or more agents the guarantee lapses and oscillation can return, though far less than with plain VO[8]. ORCA gives a half-plane velocity constraint per neighbor and picks the velocity closest to preferred inside their intersection via a small linear program (when the constraints are infeasible in a dense crowd, a three-dimensional LP picks the velocity that least violates them)[7]. RVO and ORCA are different algorithms: RVO is the averaging idea (2008), ORCA the half-plane constraints plus LP (ISRR 2009), and the reference library that implements ORCA is confusingly named RVO2[8]. RVO-family avoidance is widespread in games: Unity's navigation agents use RVO[11], and Unreal offers RVO alongside Detour's crowd manager[11].
Local avoidance does not replace pathfinding

VO/RVO/ORCA assume constant-velocity neighbors and a local horizon, so they get stuck in concave geometry and dead ends without a global path. Local avoidance is a layer on top of pathfinding, never a substitute for it.

Worked example

The widget's crossing agents are 20 px in radius (40 px combined), walk at 70 px/s, and look 1.5 s ahead. Head-on they close at 140 px/s, so their predicted collision falls inside the horizon once their centers are 40 + 140 × 1.5 = 250 px apart. At 300 px apart the predicted collision is (300 − 40) / 140 ≈ 1.9 s away, beyond the horizon, so both keep their preferred velocity. Inside 250 px each must pick a velocity outside the other's cone.

The first mode is boids with separation, cohesion, and alignment. The second has two agents crossing head-on, each choosing its velocity with a velocity obstacle; the checkbox switches between plain VO and RVO:
flocking
two crossing
reciprocal (RVO)
Pick flocking or two crossing; in two crossing, toggle the reciprocal (RVO) box and watch the trails and the heading-change count. Flocking: separation, cohesion, and alignment produce an emergent flock from local rules. Two crossing: each agent heads for the other's start and picks a new velocity six times a second. With the box unchecked (plain VO), each dodges fully, finds its way clear once the other has dodged too, snaps back, and dodges again: the trails come out as staircases and the heading-change count reaches 10 or more. With RVO, each takes half the dodge and keeps it: one smooth swerve, two heading changes. Both get past each other; the difference is the jitter.
