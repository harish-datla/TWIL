# Gameplay Effects Deep Dive

[Core Concepts](../part_2_core-concepts/01-core-concepts.md) · [Source page](https://gas.spoked.dev/gameplay-effects/)

Gameplay Effects (GEs) are the workhorse of GAS. They are the **data-driven bundles** that actually *do things* to actors: heal them, burn them, speed them up, slow them down, grant them abilities, make them immune to crowd control. If Gameplay Abilities are the verbs, Gameplay Effects are the nouns — the tangible changes applied to an actor's state.

This section goes deep. If you are looking for a high-level overview, start with the [Core Concepts: Gameplay Effects Overview](https://gas.spoked.dev/core-concepts/gameplay-effects-overview/) first, then come back here.

## The Three Duration Types (Quick Recap)

Every Gameplay Effect has a **Duration Policy** that determines its lifetime:

| Duration Policy | What It Does | Lives In Active GE Container? |
|---|---|---|
| **Instant** | Executes once, permanently changes base attribute values, then disappears | No |
| **Has Duration** | Lives on the target for a set amount of time, modifying attributes while active | Yes |
| **Infinite** | Lives on the target forever (until explicitly removed) | Yes |

Duration and Infinite effects are *added* to the Active Gameplay Effects Container. Instant effects are *executed* — they modify the base value of an attribute and are gone. This distinction matters a lot, and we cover it thoroughly in [Duration and Lifecycle](https://gas.spoked.dev/gameplay-effects/duration-and-lifecycle/).

## Decision Tree: Where Should I Go?

Not sure which page answers your question? Start here.

**"I want to..."**

- **...understand how GEs are created, live, and die** — [Duration and Lifecycle](https://gas.spoked.dev/gameplay-effects/duration-and-lifecycle/)
- **...add or multiply an attribute by some value** — [Modifiers](https://gas.spoked.dev/gameplay-effects/modifiers/)
- **...figure out how a modifier gets its number** — [Magnitude Calculations](https://gas.spoked.dev/gameplay-effects/magnitude-calculations/)
- **...write a complex formula that touches multiple attributes** — [Execution Calculations](https://gas.spoked.dev/gameplay-effects/execution-calculations/)
- **...make a buff that stacks up to 5 times** — [Stacking](https://gas.spoked.dev/gameplay-effects/stacking/)
- **...grant abilities, apply tags, or block other effects** — [GE Components](https://gas.spoked.dev/gameplay-effects/ge-components/)
- **...control when effects can apply based on tags** — [Tags and Requirements](https://gas.spoked.dev/gameplay-effects/tags-and-requirements/)
- **...make my character immune to fire damage** — [Immunity](https://gas.spoked.dev/gameplay-effects/immunity/)
- **...implement cooldowns or resource costs for abilities** — [Cooldowns and Costs](https://gas.spoked.dev/gameplay-effects/cooldowns-and-costs/)
- **...pass a damage number from code into an effect** — [SetByCaller](https://gas.spoked.dev/gameplay-effects/set-by-caller/)
- **...create or modify effects at runtime in C++** — [Dynamic Effects](https://gas.spoked.dev/gameplay-effects/dynamic-effects/)
- **...show effect names or icons in my UI** — [UI Data](https://gas.spoked.dev/gameplay-effects/ui-data/)
- **...add custom logic to control when an effect applies** — [Custom Application](https://gas.spoked.dev/gameplay-effects/custom-application/)
- **...understand the old extension system** — [Effect Extensions](https://gas.spoked.dev/gameplay-effects/effect-extensions/) (Legacy)

## Suggested Reading Order

If you are working through this for the first time, this order builds concepts progressively:

1. [Duration and Lifecycle](https://gas.spoked.dev/gameplay-effects/duration-and-lifecycle/) — the foundation
2. [Modifiers](https://gas.spoked.dev/gameplay-effects/modifiers/) — how attributes get changed
3. [Magnitude Calculations](https://gas.spoked.dev/gameplay-effects/magnitude-calculations/) — where the numbers come from
4. [SetByCaller](https://gas.spoked.dev/gameplay-effects/set-by-caller/) — the most common way to pass data into effects
5. [Execution Calculations](https://gas.spoked.dev/gameplay-effects/execution-calculations/) — the power tool for complex formulas
6. [Stacking](https://gas.spoked.dev/gameplay-effects/stacking/) — what happens when the same effect is applied twice
7. [GE Components](https://gas.spoked.dev/gameplay-effects/ge-components/) — the modular architecture (5.3+)
8. [Tags and Requirements](https://gas.spoked.dev/gameplay-effects/tags-and-requirements/) — conditional application and ongoing requirements
9. [Immunity](https://gas.spoked.dev/gameplay-effects/immunity/) — blocking effects
10. [Cooldowns and Costs](https://gas.spoked.dev/gameplay-effects/cooldowns-and-costs/) — the ability economy
11. [Dynamic Effects](https://gas.spoked.dev/gameplay-effects/dynamic-effects/) — runtime creation and modification
12. [UI Data](https://gas.spoked.dev/gameplay-effects/ui-data/) — showing effect names, descriptions, and icons in UI
13. [Custom Application](https://gas.spoked.dev/gameplay-effects/custom-application/) — custom logic controlling when effects apply
14. [Effect Extensions](https://gas.spoked.dev/gameplay-effects/effect-extensions/) — the legacy extension system (pre-5.3)

Already comfortable with GAS?

If you have shipped a project with GAS before and just need to look something up, jump straight to whatever page interests you. Each page is designed to stand on its own.

## Key Source Files

For engine spelunkers, the relevant headers live under `Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/`:

| File | What It Contains |
|---|---|
| `GameplayEffect.h` | `UGameplayEffect`, `FGameplayEffectSpec`, `FActiveGameplayEffect`, modifier structs, stacking enums |
| `GameplayEffectTypes.h` | `EGameplayModOp`, evaluation channels, `FGameplayEffectContext` |
| `GameplayEffectComponent.h` | `UGameplayEffectComponent` base class |
| `GameplayEffectComponents/*.h` | All 11 built-in component types |
| `GameplayEffectExecutionCalculation.h` | `UGameplayEffectExecutionCalculation` and its parameter/output structs |
| `GameplayModMagnitudeCalculation.h` | `UGameplayModMagnitudeCalculation` base class |
