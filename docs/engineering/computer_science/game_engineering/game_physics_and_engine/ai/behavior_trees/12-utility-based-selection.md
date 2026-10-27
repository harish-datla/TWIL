# Utility-based selection

A guard sees two enemies. A selector picks the first branch whose conditions hold, so the guard attacks whichever enemy the designer's tree happens to check first. You want it to weigh how close and how dangerous each enemy is, which takes a score rather than a fixed order.

Standard selectors pick children by fixed left-to-right priority. Utility AI replaces that with a scoring function: each child is assigned a utility value (a float computed from threat level, distance, ammo count, health, cooldown timers), and the child with the highest score wins. Dave Mark's Infinite Axis Utility System, shown briefly in a shared GDC 2013 AI Summit talk and presented in full with Mike Lewis at GDC 2015 as the rebuilt AI for Guild Wars 2, is a widely cited formulation[17].

Worked example

Score each enemy on two considerations, each from 0 to 1. Closeness is 1 − distance / 50 m, and threat comes from the enemy type. Enemy A is 10 m away (closeness 0.8) with threat 0.5. Enemy B is 30 m away (closeness 0.4) with threat 0.9. A common way to combine considerations is to multiply them, so one score near 0 vetoes the option: A scores 0.8 × 0.5 = 0.40 and B scores 0.4 × 0.9 = 0.36, so the guard engages A. Move B to 20 m (closeness 0.6) and it scores 0.54 and wins. The curves here are made up; tuning them is the real work.

A common hybrid keeps the tree for the coarse decision hierarchy (combat vs exploration vs dialogue) and uses utility scoring for the fine-grained choice within a category (which target to engage, which cover point to use). That avoids scoring everything against everything while still getting context-sensitive decisions where they matter.

Implementation: replace the Selector's left-to-right loop with a "score all children, sort by score, evaluate in score order" loop. The child evaluation still follows the standard tick protocol (SUCCESS/FAILURE/RUNNING), so the rest of the tree machinery is unchanged.
Check Your Understanding Q1 of 2

Your agent uses RUNNING to signal that a multi-frame MoveTo action is still in progress. A bug causes MoveTo to never return SUCCESS or FAILURE. What symptom does the designer see?
The agent is stuck in that branch: its sequence never advances, and nothing to its right runs.
The agent flips between behaviors every frame, because RUNNING gets treated as FAILURE.
The agent crashes once the tree detects that it has entered an infinite evaluation loop.
0/2 correct
