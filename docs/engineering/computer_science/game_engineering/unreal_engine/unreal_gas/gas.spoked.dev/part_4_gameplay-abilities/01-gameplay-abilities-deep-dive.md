# Gameplay Abilities Deep Dive

[Core Concepts](../part_2_core-concepts/01-core-concepts.md) · [Gameplay Effects Deep Dive](../part_3_gameplay-effects/01-gameplay-effects-deep-dive.md) · [Source page](https://gas.spoked.dev/gameplay-abilities/)

Gameplay Abilities are the **verbs** of GAS. They represent discrete actions an actor can perform: casting a fireball, dodging, jumping, entering a rage state, channeling a heal. Every ability goes through a well-defined lifecycle of activation, execution, and termination — and the system provides a rich set of tools for controlling each step.

If you are looking for a high-level overview, start with the [Core Concepts: Gameplay Abilities Overview](https://gas.spoked.dev/core-concepts/gameplay-abilities-overview/) first, then come back here for the deep dive.

## The Ability Lifecycle (Quick Recap)

At its core, every ability flows through the same phases:

```
CanActivateAbility()  →  ActivateAbility()  →  CommitAbility()  →  EndAbility()
       ↑                                             ↑
  Tag checks                                  Cost + Cooldown
  Cooldown check                              applied here
  Cost check
  Net role check
```

Between `ActivateAbility` and `EndAbility`, your ability is alive and running. It can spawn [Ability Tasks](https://gas.spoked.dev/gameplay-abilities/ability-tasks/) to do asynchronous work, request [targeting data](https://gas.spoked.dev/gameplay-abilities/targeting/), play montages, apply [Gameplay Effects](https://gas.spoked.dev/gameplay-effects/), and fire [Gameplay Cues](https://gas.spoked.dev/gameplay-cues/). The ability is not complete until `EndAbility` is called — and forgetting to call it is one of the most common bugs new GAS developers run into.

We cover the lifecycle in full detail on the [Lifecycle and Activation](https://gas.spoked.dev/gameplay-abilities/lifecycle-and-activation/) page.

## Decision Tree: Where Should I Go?

Not sure which page answers your question? Start here.

**"I want to..."**

- **...understand how abilities are activated, committed, and ended** — [Lifecycle and Activation](https://gas.spoked.dev/gameplay-abilities/lifecycle-and-activation/)
- **...figure out which instancing policy to use** — [Instancing Policy](https://gas.spoked.dev/gameplay-abilities/instancing-policy/)
- **...do async work inside my ability (wait for events, play montages, etc.)** — [Ability Tasks](https://gas.spoked.dev/gameplay-abilities/ability-tasks/)
- **...listen for GAS events from a UI widget or AI controller** — [Async Ability Tasks](https://gas.spoked.dev/gameplay-abilities/async-ability-tasks/)
- **...bind abilities to player input (Enhanced Input)** — [Input Binding](https://gas.spoked.dev/gameplay-abilities/input-binding/)
- **...grant a set of abilities when a character spawns or equips something** — [Ability Sets](https://gas.spoked.dev/gameplay-abilities/ability-sets/)
- **...implement interactive targeting (aim at the ground, select actors, etc.)** — [Targeting](https://gas.spoked.dev/gameplay-abilities/targeting/) (sub-pages: [Target Data](https://gas.spoked.dev/gameplay-abilities/targeting/target-data/), [Target Actors](https://gas.spoked.dev/gameplay-abilities/targeting/target-actors/), [Reticles and Filters](https://gas.spoked.dev/gameplay-abilities/targeting/reticles-and-filters/))
- **...see what the built-in example abilities look like** — [Built-in Abilities](https://gas.spoked.dev/gameplay-abilities/built-in-abilities/)

**"I have a problem with..."**

- **...my ability never ending / leaking** — Check the [EndAbility section](https://gas.spoked.dev/gameplay-abilities/lifecycle-and-activation/#endability) and make sure every code path calls it.
- **...tasks crashing when the ability ends** — Check your [Instancing Policy](https://gas.spoked.dev/gameplay-abilities/instancing-policy/) — you may be using NonInstanced with tasks (don't do that).
- **...abilities not activating** — Walk through the [full activation check sequence](https://gas.spoked.dev/gameplay-abilities/lifecycle-and-activation/#the-full-check-sequence) to see which check is failing. Use `ShowDebug AbilitySystem` to inspect tags.
- **...input not working with abilities** — See [Input Binding](https://gas.spoked.dev/gameplay-abilities/input-binding/) for the Enhanced Input integration patterns.

## Suggested Reading Order

If you are working through this section for the first time, this order builds concepts progressively:

1. [Lifecycle and Activation](https://gas.spoked.dev/gameplay-abilities/lifecycle-and-activation/) — the foundation, how abilities live and die
2. [Instancing Policy](https://gas.spoked.dev/gameplay-abilities/instancing-policy/) — how abilities are allocated in memory (affects everything else)
3. [Ability Tasks](https://gas.spoked.dev/gameplay-abilities/ability-tasks/) — the async building blocks for ability logic
4. [Async Ability Tasks](https://gas.spoked.dev/gameplay-abilities/async-ability-tasks/) — the newer alternative for non-ability contexts
5. [Input Binding](https://gas.spoked.dev/gameplay-abilities/input-binding/) — connecting player input to ability activation
6. [Ability Sets](https://gas.spoked.dev/gameplay-abilities/ability-sets/) — organizing and granting groups of abilities
7. [Targeting](https://gas.spoked.dev/gameplay-abilities/targeting/) — the full targeting subsystem
8. [Built-in Abilities](https://gas.spoked.dev/gameplay-abilities/built-in-abilities/) — learning from Epic's examples

Already comfortable with GAS?

If you have shipped a project with GAS before and just need to look something up, jump straight to whatever page interests you. Each page is designed to stand on its own.

## Key Source Files

For engine spelunkers, the relevant headers live under `Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/`:

| File | What It Contains |
|---|---|
| `Abilities/GameplayAbility.h` | `UGameplayAbility` — the full ability class with lifecycle, commit, cancel, tags, cues |
| `GameplayAbilitySpec.h` | `FGameplayAbilitySpec`, `FGameplayAbilityActivationInfo`, `FGameplayAbilitySpecDef` |
| `Abilities/GameplayAbilityTypes.h` | `FGameplayAbilityActorInfo`, instancing/net execution/trigger enums |
| `Abilities/Tasks/AbilityTask.h` | `UAbilityTask` base class |
| `Abilities/Tasks/AbilityTask_*.h` | All 27+ built-in ability tasks |
| `Abilities/Async/AbilityAsync.h` | `UAbilityAsync` base class for async tasks |
| `Abilities/Async/AbilityAsync_*.h` | All 6 async ability task types |
| `Abilities/GameplayAbilityTargetActor*.h` | Target actor classes for interactive targeting |
| `Abilities/GameplayAbilityTargetTypes.h` | `FGameplayAbilityTargetData` and its handle |
| `Abilities/GameplayAbilityTargetDataFilter.h` | `FGameplayTargetDataFilter` |
| `Abilities/GameplayAbilityWorldReticle*.h` | World reticle actors for targeting visualization |
| `GameplayAbilitySet.h` | `UGameplayAbilitySet` data asset |
| `Abilities/GameplayAbility_CharacterJump.h` | Built-in jump ability example |
| `Abilities/GameplayAbility_Montage.h` | Built-in montage ability example |
