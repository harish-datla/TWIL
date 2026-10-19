# Pitfalls

    Deep trees that are hard to read. Ten levels of nesting read about as well as spaghetti code. Keep the tree shallow and extract reusable subtrees into named sub-behaviors; Unreal lets a tree run another tree asset as a subtree, which keeps each graph manageable.
    Over-use of Parallel. Running multiple behaviors simultaneously creates interaction bugs. If a Parallel ticks both "move to cover" and "fire at enemy," the agent may try to shoot while running, producing broken animations. Use Parallel sparingly and test the interaction between concurrent children explicitly.
    Blackboard key collisions. Two subtrees write the same blackboard key ("target") without coordination: to the cover subtree it is a cover point, to the combat subtree an enemy. Fix: use namespaced keys ("combat.target", "cover.target") or separate blackboard scopes per subtree.
    Forgetting to handle RUNNING. An action that returns RUNNING but never transitions to SUCCESS or FAILURE blocks its branch forever (see the quiz in section 12). Every action that returns RUNNING must have a termination condition. Add TimeLimit decorators as safety nets.
    Priority starvation. A high-priority branch that succeeds every tick prevents lower-priority branches from ever running. If "CheckForEnemies" always succeeds (even when there's nothing to do about the enemy), everything below it starves. Conditions should return FAILURE when their associated behavior is not actionable, not just when the world state doesn't match.
    Condition/action ordering in sequences. Put conditions before actions. A sequence [Fire, HasAmmo?] fires first and checks ammo after, which is backwards. The tree reads left to right, so preconditions go on the left.

