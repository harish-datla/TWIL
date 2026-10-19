# Cheat Sheet


A page you can paste into your engine's contributing guide. For each allocator: what sizes, what lifetimes, what worst-case time, what waste, and where it ships.
Allocator 	Sizes 	Lifetimes 	Worst-case 	Waste 	Ship example
Linear / bump 	Any 	Bulk-free only 	O(1) 	Alignment padding 	UE5 FMemStack
Stack (LIFO markers) 	Any 	Nested scopes 	O(1) 	Alignment padding 	Frostbite scope-stack[28]
Pool (fixed-size) 	One size 	Any per-object 	O(1) 	Internal only (slot vs payload) 	Particle pools, bullet pools
Buddy 	Powers of two 	Any per-object 	O(log heap) 	Internal, ~25–30%[7]; plus external 	Linux page allocator
TLSF 	Any 	Any per-object 	O(1), ≤160 instr 	< 1/2SLI per alloc 	Unity dynamic heap, VMA default
Slab 	One size per cache 	Any per-object 	O(1) on magazine 	Slab overhead + internal 	Solaris kmem, Linux SLUB
Free-list (first-fit) 	Any 	Any per-object 	O(n) free blocks 	External fragmentation 	K&R malloc (a next-fit variant), small embedded heaps

[Back to Memory Allocators from Scratch](index.md)
