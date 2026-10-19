# Decorators

A guard shouts "Over there!" when it spots you. Left alone, the bark's condition stays true while you're in view, so the guard starts a new bark the moment the last one ends. You don't want to edit the Bark action; you want to wrap it in "after each bark, stay quiet for ten seconds". That wrapper is a decorator.

A decorator wraps exactly one child and modifies its behavior or result. Common decorators:

    Inverter (NOT). Flips SUCCESS to FAILURE and vice versa. RUNNING passes through. Useful for negating conditions: "if NOT EnemyVisible, patrol."
    Repeat(n). Ticks its child up to n times. Returns SUCCESS after n completions, RUNNING while iterating, FAILURE if the child fails during any iteration.
    RetryUntilSuccess(n). Retries the child up to n times on FAILURE. Returns SUCCESS as soon as the child succeeds. Returns FAILURE after n failed attempts.
    Cooldown(t). After the child completes (SUCCESS or FAILURE), returns FAILURE for the next t seconds without ticking the child. Prevents behaviors from re-triggering immediately (barks, reload checks, perception queries).
    TimeLimit(t). If the child has been RUNNING for more than t seconds, forces a FAILURE. Prevents stuck actions from blocking the tree forever.

Worked example

Wrap a three-second Bark in Cooldown(10). The guard spots you at t = 0 s and the bark plays until t = 3 s. From 3 s to 13 s the decorator returns FAILURE without ticking Bark, so the parent selector moves on to the next branch and the guard gets on with chasing you. At t = 13 s, if you're still in view, it barks again: one bark every 13 seconds instead of back-to-back barks.

Unreal uses decorators differently. They attach to a composite or task node rather than sitting in the tree as a parent of their own, and Epic recommends them in place of condition leaves: "In the standard model for Behavior Trees, conditionals are Task leaf nodes", whereas Unreal's conditional decorators gate a branch so that every leaf is an action[6][22]. Those decorators double as abort triggers: when a decorator's condition changes, it can abort its own subtree, the lower-priority branches to its right, or both (§11).
