# Debugging GAS

[Examples](../part_9_examples/01-examples.md) · [Source page](https://gas.spoked.dev/debugging/)

Something's not working. Let's figure out what.

## Triage: What's Going Wrong?

Pick the symptom that matches your situation.

### My Ability Won't Activate

1. **Is the ability granted?** Open the console and run `showdebug abilitysystem`. Look at the **Granted Abilities** list. Is your ability there?
   - **Not listed** — The ability was never granted. Check that it's in your character's `StartupAbilities` array, or that `GiveAbility()` is being called at runtime.
   - **Listed** — Continue to the next check.
2. **Is it blocked by tags?** Check the owner's active tags and the ability's activation required and blocked tags. Use `showdebug abilitysystem` to inspect tags.
3. **Is it on cooldown?** Check whether the ability's cooldown Gameplay Effect is active and whether its cooldown tags are present.
4. **Can the owner afford the cost?** Check the relevant attributes and the ability's cost Gameplay Effect.
5. **Is the Net Execution Policy correct?** Confirm the policy matches the intended activation path and network role.
6. **Check the output log** for activation failure messages, missing assets, or tag warnings.

### My Effect Doesn't Modify the Attribute

1. **Is the effect actually applied?** Confirm the target has an Ability System Component and inspect its active effects with `showdebug abilitysystem`.
2. **Is the modifier targeting the right attribute?** Check the modifier's Attribute and operation.
3. **Is the magnitude non-zero?** Verify the configured magnitude, any SetByCaller value, and any magnitude calculation inputs.
4. **Are tag requirements filtering the modifier?** Check source and target tags against the effect's application and ongoing requirements.
5. **Is the attribute on the target's AttributeSet?** Confirm the target has the AttributeSet that defines the modified attribute.

### My Cue Doesn't Play

1. **Does the tag match?** Check that the Gameplay Cue tag on the effect or ability matches the tag on the cue notify.
2. **Is the cue discovered?** Confirm its asset path is included in the Gameplay Cue Manager's scan paths and that the cue notify map contains it.
3. **Is the cue loaded?** Check the configured loading behavior and the [lazy loading](https://gas.spoked.dev/optimization/lazy-loading/) setup.
4. **Are cues suppressed?** Check whether gameplay cues are suppressed for the actor, Ability System Component, or current execution context.

## Debugging Tools

| Tool | Best For | Details |
|---|---|---|
| **ShowDebug AbilitySystem** | Real-time HUD overlay of abilities, effects, attributes, and tags | [Details](https://gas.spoked.dev/debugging/showdebug-abilitysystem/) |
| **Gameplay Debugger** | Inspecting any actor in the world (not just the player) | [Details](https://gas.spoked.dev/debugging/gameplay-debugger/) |
| **Output Log** | Activation failures, missing SetByCaller values, and tag warnings | [Details](https://gas.spoked.dev/debugging/logging/) |
| **Troubleshooting FAQ** | Specific symptoms with known causes and fixes | [Details](https://gas.spoked.dev/debugging/troubleshooting/) |
| **Testing GAS** | Automated testing patterns for abilities and effects | [Details](https://gas.spoked.dev/debugging/testing-gas/) |

## Quick Console Commands

| Command | What It Does |
|---|---|
| `showdebug abilitysystem` | Toggle the ability system debug HUD |
| `AbilitySystem.Debug.NextCategory` | Cycle through debug pages (attributes, abilities, effects) |
| `AbilitySystem.Debug.NextTarget` | Cycle through debug targets |
| `AbilitySystem.GameplayCue.PrintGameplayCueNotifyMap` | Dump all registered cue-to-class mappings |
| `Log LogAbilitySystem Verbose` | Enable verbose ability logging |
| `Log LogGameplayEffects Verbose` | Enable verbose effect logging |
