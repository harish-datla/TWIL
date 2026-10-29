# Patterns

[Core Concepts](../part_2_core-concepts/01-core-concepts.md) · [Source page](https://gas.spoked.dev/patterns/)

This section covers the design patterns that experienced GAS developers have landed on after shipping games. These aren't the only way to do things, but they represent well-tested approaches that work across a range of project types.

## Patterns vs Recipes

**Patterns** explain the *design* — the why, the trade-offs, and the architecture. [Recipes](https://gas.spoked.dev/recipes/) are the step-by-step *procedures* for implementing specific things. Patterns link to recipes where applicable.

## In This Section

### [Common Abilities](https://gas.spoked.dev/patterns/common-abilities/)

Implementation patterns for abilities that show up in nearly every action game: stun/CC, sprint, aim-down-sights, lifesteal, critical hits, interaction systems, passive abilities, and toggle abilities.

### [Damage Pipeline](https://gas.spoked.dev/patterns/damage-pipeline/)

The end-to-end flow of damage from source to target: GE spec creation, ExecCalc with armor/resistance/crits, meta attributes, `PostGameplayEffectExecute` for health subtraction and death checks. Full code example included.

### [Buff/Debuff System](https://gas.spoked.dev/patterns/buff-debuff-system/)

Duration effects with tags, stacking policies, UI representation, cleansing/dispelling, buff categories, and immunity to specific debuff types.

### [Tag Architecture](https://gas.spoked.dev/patterns/tag-design/)

Namespace design principles, a 21-namespace starter preset, loading tags from `.ini` files, and governance strategies for teams.

### [C++ vs Blueprint](https://gas.spoked.dev/patterns/cpp-vs-blueprint/)

What must be C++, what should be C++, what works best in Blueprint, and how the split changes with project scale. Includes a table mapping every GAS asset type to its recommended creation location.

### [Naming Conventions](https://gas.spoked.dev/patterns/naming-conventions/)

Prefix conventions for assets (GA_, GE_, GC_), class naming (A/U prefixes), tag naming patterns, and Content Browser folder structure.

### [Connecting GAS to UI](https://gas.spoked.dev/patterns/gas-to-ui/)

Connecting GAS to UI — attribute listeners, cooldown displays, buff icons, damage numbers.

### [Starter Tag Preset](https://gas.spoked.dev/patterns/starter-tags/)

Starter Tag Preset — 135+ tags across 21 namespaces, ready to copy.

### [Starter Attributes](https://gas.spoked.dev/patterns/starter-attributes/)

Starter Attributes — 50+ attributes with base values and C++ declarations.
