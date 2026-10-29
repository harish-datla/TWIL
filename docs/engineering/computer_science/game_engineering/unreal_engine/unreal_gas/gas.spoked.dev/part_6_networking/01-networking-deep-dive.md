# Networking Deep Dive

[Core Concepts](../part_2_core-concepts/01-core-concepts.md) · [Source page](https://gas.spoked.dev/networking/)

GAS has a sophisticated networking layer that handles replication of abilities, effects, attributes, tags, and cues across server and clients. The good news: for many games, the defaults work fine and you don't need to think about most of this. The bad news: when you do need to think about it, it's one of the more complex parts of the system.

## Do You Need This Section?

Be honest about your project's needs:

| Situation | What To Read |
|---|---|
| **Single-player game** | You can skip this entire section. Set replication mode to `Minimal` and move on. |
| **Co-op (2-4 players)** | Read [Replication Modes](https://gas.spoked.dev/networking/replication-modes/) and [Net Execution Policies](https://gas.spoked.dev/networking/net-execution-policies/). The defaults will mostly work. |
| **Competitive multiplayer (4-16 players)** | Read everything here. Prediction and execution policies matter a lot for game feel. |
| **Large-scale (32+ players, MMO-like)** | Read everything here plus the [Optimization section](https://gas.spoked.dev/optimization/). You'll need Minimal replication and the proxy interface. |
| **Dedicated server** | You need this. Pay special attention to prediction and execution policies. |
| **Listen server** | Simpler than dedicated, but still read replication modes and policies. |

## GAS Networking Philosophy

A few core principles to understand before diving in:

**The server is authoritative.** The server owns the truth about gameplay state. Abilities activate on the server, effects are applied on the server, and the results replicate down to clients. Client prediction is an optimization, not a source of truth.

**Prediction is opt-in and scoped.** Not everything is predicted. Ability activation, GE application (attribute mods and tags), gameplay cues, and montages are predicted. Execution Calculations, GE removal, and periodic ticks are not. Prediction happens within a window tied to an ability activation.

**Replication mode is a bandwidth dial.** GAS provides three replication modes (Full, Mixed, Minimal) that control how much data flows to which clients. Picking the right one is the single biggest networking decision you'll make.

**Cues are separate from gameplay state.** Cue events are sent through their own RPC path, independent of attribute/effect replication. This allows them to be batched, throttled, or made unreliable without affecting gameplay correctness.

## What's in This Section

### [Replication Modes](https://gas.spoked.dev/networking/replication-modes/)

The three modes (Full, Mixed, Minimal) and what each replicates to whom. This is the first thing to configure for your project.

### [Prediction](https://gas.spoked.dev/networking/prediction/)

`FPredictionKey`, prediction windows, what gets predicted, misprediction rollback, and the common pitfalls that trip people up.

### [Net Execution Policies](https://gas.spoked.dev/networking/net-execution-policies/)

The four policies (LocalPredicted, LocalOnly, ServerInitiated, ServerOnly) that control where abilities execute and how they interact with replication.

### [Batching](https://gas.spoked.dev/networking/batching/)

Ability activation batching, cue batching, and `FScopedServerAbilityRPCBatcher` for reducing RPC overhead.

### [Replication Proxy](https://gas.spoked.dev/networking/replication-proxy/)

`IAbilitySystemReplicationProxyInterface` for moving replication off the ASC and onto a proxy actor, plus the net serializers in the `Serialization/` folder.

## Key Concepts Quick Reference

| Concept | What It Is |
|---|---|
| **FPredictionKey** | Unique ID generated client-side, sent to server, used to match predicted and authoritative state |
| **Prediction Window** | The scope during which predicted side effects are valid (typically the initial ActivateAbility callstack) |
| **EGameplayEffectReplicationMode** | Full/Mixed/Minimal -- controls GE replication bandwidth |
| **EGameplayAbilityNetExecutionPolicy** | LocalPredicted/LocalOnly/ServerInitiated/ServerOnly -- controls where ability code runs |
| **ActiveGameplayCueContainer** | Fast array serializer for replicated duration cues |
| **FMinimalGameplayCueReplicationProxy** | Lightweight cue replication for Minimal mode |
| **FScopedServerAbilityRPCBatcher** | Batches ability activation RPCs |

## Key Source Files

| File | What's Inside |
|---|---|
| `GameplayPrediction.h` | `FPredictionKey`, prediction architecture documentation |
| `AbilitySystemComponent.h` | Replication mode enum, RPC declarations |
| `GameplayCueInterface.h` | `FActiveGameplayCueContainer`, `FMinimalGameplayCueReplicationProxy` |
| `AbilitySystemReplicationProxyInterface.h` | `IAbilitySystemReplicationProxyInterface` |
| `Serialization/*.h` | Net serializers for various GAS types |
| `Abilities/GameplayAbility.h` | Net execution policy enum |
