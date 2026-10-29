# Optimization

[Networking Deep Dive](../part_6_networking/01-networking-deep-dive.md) · [Source page](https://gas.spoked.dev/optimization/)

## Only Read This After Your Game Works

Premature optimization is the root of all evil. Get your abilities, effects, and cues working correctly first. Then profile. Then optimize what the profiler says is slow. Don't guess.

If you're here because you've profiled and identified GAS as a bottleneck (or you're planning for large player counts), this section covers the main optimization levers.

## What to Optimize, and When

| Symptom | Likely Cause | Page |
|---|---|---|
| High bandwidth per player | Too much GE replication | [Replication Optimization](https://gas.spoked.dev/optimization/replication-optimization/) |
| RPC saturation / dropped cues | Too many cue RPCs | [Cue Batching](https://gas.spoked.dev/optimization/cue-batching/) |
| Bandwidth scales badly with player count | Need minimal replication + proxy | [Attribute Proxy](https://gas.spoked.dev/optimization/attribute-proxy/) |
| Hitches when cues first play | Sync loading cue notifies | [Lazy Loading](https://gas.spoked.dev/optimization/lazy-loading/) |
| Long initial load times | Loading all cues at startup | [Lazy Loading](https://gas.spoked.dev/optimization/lazy-loading/) |

## The Optimization Hierarchy

In rough order of impact and ease of implementation:

1. **Choose the right replication mode.** Mixed for most games, Minimal for large-scale. This is the biggest single win.
2. **Batch cue RPCs.** Use `FScopedGameplayCueSendContext` whenever applying multiple effects.
3. **Make cues unreliable.** They already are by default -- don't override to reliable unless you have a specific need.
4. **Reduce active effect count.** Combine effects where possible. An effect with 3 modifiers is cheaper than 3 effects with 1 modifier each.
5. **Pre-allocate cue actors.** Set `NumPreallocatedInstances` on frequently used Actor/Looping cues to avoid runtime spawning.
6. **Async load cues.** Don't sync-load missing cues at runtime. Configure the manager for async loading.
7. **Implement the replication proxy.** For 50+ player games, move replication to a proxy actor with better relevancy control.

## Profiling GAS

### Network Profiler

The built-in network profiler shows RPC counts and bandwidth per actor. Look for:

- `GameplayCue` RPCs (should be batched, unreliable)
- `AbilitySystem` RPCs (activation, prediction)
- `FActiveGameplayEffect` replication bandwidth

### Stat Commands

```text
stat net                    // General network stats
stat game                   // Game thread time
stat AbilitySystem          // GAS-specific stats (if available)
```

### Unreal Insights

Use Unreal Insights for detailed CPU profiling of GAS operations:

- Effect application and removal
- Modifier evaluation
- Cue routing and spawning
- Attribute aggregation

## In This Section

- [Replication Optimization](https://gas.spoked.dev/optimization/replication-optimization/) -- choosing and configuring replication modes
- [Cue Batching](https://gas.spoked.dev/optimization/cue-batching/) -- reducing cue RPC overhead
- [Attribute Proxy](https://gas.spoked.dev/optimization/attribute-proxy/) -- the replication proxy for large player counts
- [Lazy Loading](https://gas.spoked.dev/optimization/lazy-loading/) -- async loading cue notifies and ability classes
