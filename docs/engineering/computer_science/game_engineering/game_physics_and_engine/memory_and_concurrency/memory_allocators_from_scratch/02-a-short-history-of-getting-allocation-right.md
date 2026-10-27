# A Short History of Getting Allocation Right


Why does Linux still hand out physical pages with a 1965 algorithm, and why does Unity's heap use an allocator from a 2004 real-time systems paper? Because the C library's malloc didn't arrive fully formed, and neither did the specialized allocators engines use. Most of them trace back to a short list of papers and talks.
Timeline: from the buddy system (1963) to Linux 6.8 (2024)

Three results from that history matter for the rest of the page. Any allocator that can't move live blocks can be fragmented by a bad enough request pattern (Robson, 1971[12]). Real programs fragment far less than that worst case, largely because objects allocated together tend to die together (Wilson et al., 1995, and Johnstone and Wilson, 1998[1][14]). And most of the fast allocators engines use give up some generality to win speed, as Knowlton's buddy system, Hanson's arenas and Bonwick's slabs do; TLSF keeps the generality and bounds the worst case instead.

[Back to Memory Allocators from Scratch](index.md)
