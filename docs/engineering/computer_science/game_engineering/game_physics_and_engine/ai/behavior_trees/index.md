# Behavior Trees

Behavior trees organize NPC decisions as prioritized branches of actions and conditions. This tutorial builds a runtime with success, failure, and running statuses, then covers composites, decorators, blackboards, reactive behavior, and alternatives such as utility AI, GOAP, and HTN.

Time: ~45 min  
Level: Junior to mid  
Prerequisites: C++ or pseudocode, trees, recursion, and basic game-loop awareness.  
Phase: 11 · AI

## What you’ll be able to do

- Build sequences and selectors from nodes returning `SUCCESS`, `FAILURE`, or `RUNNING`.
- Read and prioritize a behavior tree tick by tick.
- Share perception and state through per-agent blackboards.
- Choose memory, reactive, or abort-driven behavior for responsive NPCs.
- Know when utility scoring, GOAP, or HTN fits better than a hand-built tree.

## Sections

- [Why Behavior Trees](01-why-behavior-trees.md)
- [A Short History](02-a-short-history.md)
- [The FSM Problem](03-the-fsm-problem.md)
- [Tree Structure and Tick](04-tree-structure-and-tick.md)
- [Sequences](05-sequences.md)
- [Selectors](06-selectors.md)
- [Decorators](07-decorators.md)
- [Parallel Nodes](08-parallel-nodes.md)
- [Blackboards](09-blackboards.md)
- [Actions and Conditions](10-actions-and-conditions.md)
- [Memory and Reactive Behavior](11-memory-and-reactive-behavior.md)
- [Utility-Based Selection](12-utility-based-selection.md)
- [BT vs GOAP vs HTN](13-bt-vs-goap-vs-htn.md)
- [Case Studies](14-case-studies.md)
- [Pitfalls](15-pitfalls.md)
- [What’s Next](16-whats-next.md)
