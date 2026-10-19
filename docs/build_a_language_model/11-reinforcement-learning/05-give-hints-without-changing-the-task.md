# Give hints without changing the task

In the grid, the only positive reward is at the goal, so a learner wanders for a long time before anything useful happens. It's tempting to add hints, like a small bonus for each move toward the goal. Careless bonuses change the task, though. Reward a soccer robot for touching the ball and it can learn to stand next to the ball tapping it over and over instead of scoring. Potential-based shaping is a way to add hints that provably doesn't change which policy is best.[2]

Give every state a fixed number, its potential Φ (phi), that says how promising it looks. Here the potential is minus the number of moves to the goal. The bonus for a move is γ times the potential where you land, minus the potential where you started. Moving toward the goal earns a bonus and backing away costs one. Over a complete episode the bonuses cancel down to one number that depends only on where you started, so every complete route gets the same extra amount.

Worked example

Take the three-move route from square 6 again (6 → 7 → 11 → 15) with γ = 0.9. The potentials along it are −3, −2, −1 and 0; the goal's potential is always 0.

Bonuses: 0.9 × (−2) − (−3) = 1.2, then 0.9 × (−1) − (−2) = 1.1, then 0.9 × 0 − (−1) = 1.

Discounted total: 1.2 + 0.9 × 1.1 + 0.9² × 1 = 1.2 + 0.99 + 0.81 = 3, which is exactly −Φ(6). Any other complete route from square 6 gets the same 3.

As a formula, the bonus is computed for each move and added to the real reward:
shaping(s,s′) = γ × Φ(s′) − Φ(s)
shaped_reward = original_reward + shaping(s,s′)

In words: the shaping bonus for moving from s to s′ is γ times the new state's potential minus the old state's potential, and the agent trains on the real reward plus that bonus. Use the same γ as the return, and give the goal a potential of zero.

Why the bonuses cancel: add them up along any route of T moves, with discounting, and every potential in the middle of the route appears once with a plus sign and once with a minus sign. All that's left is −Φ(start) + γTΦ(end). With the goal's potential at zero, the extra return is just −Φ(start), the same for every complete route from that start, so the ranking of routes from best to worst can't change.[2]
Tasks that never end

Check that shaping shifts every route equally
Discount γ
0.9
Potential scale
1
Reset
Paths of six and eight steps. Base returns 0.426686 / 0.269616; shaped 6.426686 / 6.269616. Added returns 6.000000 / 6.000000 = −Φ(start); Φ(goal)=0.

Drag γ or the potential scale and compare two complete routes from square 0 to the goal. Path A is the six-move shortest route; Path B takes eight moves, including a step back. Each panel shows a path's real return and its shaped return. At the defaults, A's returns are 0.4267 and 6.4267 and B's are 0.2696 and 6.2696: shaping adds exactly 6, which is −Φ(start), to both, so A stays ahead by the same margin. Changing γ changes the returns but both paths still get the same amount added. Set the scale to 0 to turn shaping off.

Path A visits squares 0 → 1 → 2 → 3 → 7 → 11 → 15. Path B visits 0 → 4 → 8 → 9 → 8 → 9 → 10 → 14 → 15. The potential is minus the Manhattan distance to the goal (rows apart plus columns apart), times the scale slider. It ignores the wall, and the cancellation still works, because the argument holds for any fixed potential; a poor potential just gives less useful hints. A potential that changes during training isn't covered by this argument.

So shaping can help a learner find a good policy faster without changing which policy is best. It can't fix a reward that measures the wrong thing, and it can't supply information the state is missing.
