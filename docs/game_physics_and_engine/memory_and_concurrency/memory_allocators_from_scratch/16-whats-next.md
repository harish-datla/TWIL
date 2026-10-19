# What's Next


Next in the series: Job Systems. It builds a scheduler that spreads a frame's work across every core, from one thread up to dependency tracking and work stealing, and it's the setting for the job-migration pitfall above. Beyond that, three directions.

Read the surveys. Wilson, Johnstone, Neely, Boles 1995[1] is still the standard survey of the field, history included; Johnstone & Wilson 1998[14] is the measurement follow-up. The Jones, Hosking, Moss Garbage Collection Handbook[45] covers the other half of the picture: collectors that can move objects, and so can compact away the fragmentation a native heap has to live with.

Read the talks and papers. Gyrling 2015[24] and MacDougall 2016[35] are two GDC talks on shipped console memory systems; Acton's CppCon 2014 talk[38] covers the data-layout side; Pedriana's EASTL paper[19] covers what game code needs from a container library's allocator interface.

Read the source. Matt Conte's TLSF[33] is a complete production allocator in about 1,300 lines of C; the VMA repository has its TLSF implementation alongside a linear algorithm for ring-buffer use[18]; Bevy ECS[2] and EnTT[3] both implement generational entity IDs in the open. The original Doom 3 idHeap[44] is a rare case of a shipped game's allocator published with the rest of its source.

[Back to Memory Allocators from Scratch](index.md)
