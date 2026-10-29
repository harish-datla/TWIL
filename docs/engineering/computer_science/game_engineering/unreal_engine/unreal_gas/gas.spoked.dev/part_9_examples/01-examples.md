# Examples

[Recipes](../part_8_recipes/01-recipes.md) · [Source page](https://gas.spoked.dev/examples/)

Complete, end-to-end ability implementations. Each example walks through the full GAS loop — effects, ability blueprint, input wiring, and testing — so you can follow along from scratch or use them as reference for your own abilities.

Every example is **standalone**. You can jump to any one without reading the others.

## Looking for a Simpler Starting Point?

The [Getting Started](https://gas.spoked.dev/getting-started/) section includes a [melee attack tutorial](https://gas.spoked.dev/getting-started/your-first-ability/) — a simplified first ability that covers the full GAS loop. Start there if you're new, then come back here for production-ready versions.

## The Examples

### Beginner

| Example | What It Demonstrates |
|---|---|
| [Jump](https://gas.spoked.dev/examples/jump/) | Stamina cost, cooldown, variable jump height via WaitInputRelease, tag-based CC blocking. |
| [Sprint Toggle](https://gas.spoked.dev/examples/sprint/) | Toggle/held ability, infinite-duration effects, periodic stamina drain, WaitInputRelease. |
| [Stamina Regen](https://gas.spoked.dev/examples/stamina-regen/) | Pure GE + tag system -- passive regen with delay-after-use, Ongoing Tag Requirements, no ability code. |

### Intermediate

| Example | What It Demonstrates |
|---|---|
| [Dodge Roll](https://gas.spoked.dev/examples/dodge-roll/) | Stamina cost, cooldown, i-frames via Activation Owned Tags, root motion, ability blocking. |
| [Melee Attack](https://gas.spoked.dev/examples/melee-attack/) | Animation-driven hit timing, SetByCaller damage, montage lifecycle, gameplay events. |
| [Ranged Attack](https://gas.spoked.dev/examples/ranged-attack/) | Projectile spawning, passing GE specs to actors, mana cost, aim direction, travel-time damage. |
| [Passive Aura](https://gas.spoked.dev/examples/passive-aura/) | Passive activation, periodic area scanning, applying effects to other actors' ASCs, handle tracking. |

### Advanced

| Example | What It Demonstrates |
|---|---|
| [Custom Damage Calculation](https://gas.spoked.dev/examples/exec-calc-damage/) | UGameplayEffectExecutionCalculation, attribute captures, elemental resistance, crit formula. C++ only. |
| [Network-Predicted Ability](https://gas.spoked.dev/examples/predicted-ability/) | Client-side prediction, predict-confirm-reject cycle, FScopedPredictionWindow, multiplayer. C++ only. |

## What You Need

All examples assume you've completed [Project Setup](https://gas.spoked.dev/getting-started/project-setup/) and have:

- A character with an **Ability System Component**
- An **AttributeSet** with at minimum Health, Stamina, and Mana
- A **base ability class** (`YourProjectGameplayAbility`) with an InputTag property
- An **input binding system** that routes Enhanced Input actions to abilities by tag

If any of that is missing, start with [Project Setup](https://gas.spoked.dev/getting-started/project-setup/).
