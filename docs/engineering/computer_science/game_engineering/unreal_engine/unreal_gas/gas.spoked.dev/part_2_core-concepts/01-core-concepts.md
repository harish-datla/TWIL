# Core Concepts

[Getting Started](../part_1_getting-Started/02-getting-started.md) · [Source page](https://gas.spoked.dev/core-concepts/)

This section breaks down each of the six pieces of GAS in depth. The [Mental Model](https://gas.spoked.dev/getting-started/mental-model/) introduced them at a high level — here, you'll learn how each one actually works, what its API looks like, and where the gotchas hide.

## The Six Pieces

Every GAS feature is built from some combination of these:

| Piece | Page | Role |
|---|---|---|
| **Ability System Component** | [ASC](https://gas.spoked.dev/core-concepts/ability-system-component/) | The hub — owns and orchestrates everything else |
| **Gameplay Tags** | [Tags](https://gas.spoked.dev/core-concepts/gameplay-tags/) | The language — hierarchical labels that drive all GAS decisions |
| **Attributes & Attribute Sets** | [Attributes](https://gas.spoked.dev/core-concepts/attributes-and-attribute-sets/) | The stats — numeric values like Health, Mana, Armor |
| **Gameplay Effects** | [Effects](https://gas.spoked.dev/core-concepts/gameplay-effects-overview/) | The pipeline — the only way to modify attributes during gameplay |
| **Gameplay Abilities** | [Abilities](https://gas.spoked.dev/core-concepts/gameplay-abilities-overview/) | The actions — discrete things a character can do |
| **Gameplay Cues** | [Cues](https://gas.spoked.dev/core-concepts/gameplay-cues-overview/) | The feedback — VFX and SFX, cosmetic only |

## How They Depend on Each Other

These pieces aren't independent — they build on each other. Understanding the dependency chain helps you learn them in the right order.

- **Ability System Component** — *The Hub*
- **Gameplay Tags** — *The Language*
- **Attributes & Attribute Sets** — *The Stats*
- **Gameplay Effects** — *The Pipeline*
- **Gameplay Abilities** — *The Actions*
- **Gameplay Cues** — *The Feedback*

Reading the diagram bottom-up:

- **Effects** depend on **Attributes** (they modify stats), **Tags** (they use tags for requirements and grants), and trigger **Cues** (for visual feedback).
- **Abilities** depend on **Effects** (they apply effects for costs, cooldowns, and results) and **Tags** (for activation rules).
- **Cues** depend on **Tags** (they're routed by GameplayCue.* tags) and are triggered by **Effects**.
- Everything flows through the **ASC**.

## Suggested Reading Order

You can read these pages in any order, but this sequence minimizes "wait, what's that?" moments:

1. **[Ability System Component](https://gas.spoked.dev/core-concepts/ability-system-component/)** — Start here. It's the hub that owns everything else, so understanding it first gives you context for the rest.
2. **[Gameplay Tags](https://gas.spoked.dev/core-concepts/gameplay-tags/)** — Tags are the language everything speaks. You'll see them referenced in every other page.
3. **[Attributes and Attribute Sets](https://gas.spoked.dev/core-concepts/attributes-and-attribute-sets/)** — Your stats. Effects modify these, so learn what they are before learning how to change them.
4. **[Data-Driven Attributes](https://gas.spoked.dev/core-concepts/data-driven-attributes/)** — How to initialize and scale attributes from DataTables, CurveTables, and Gameplay Effects instead of hardcoding values.
5. **[Gameplay Effects](https://gas.spoked.dev/core-concepts/gameplay-effects-overview/)** — The pipeline that modifies attributes, grants tags, and triggers cues.
6. **[Gameplay Abilities](https://gas.spoked.dev/core-concepts/gameplay-abilities-overview/)** — The actions your characters perform. They use effects, tags, and attributes.
7. **[Gameplay Cues](https://gas.spoked.dev/core-concepts/gameplay-cues-overview/)** — Visual and audio feedback. Depends on effects and tags.

After this section, the [Deep Dives](https://gas.spoked.dev/gameplay-effects/) go into exhaustive detail on Effects, Abilities, Cues, and Networking.
