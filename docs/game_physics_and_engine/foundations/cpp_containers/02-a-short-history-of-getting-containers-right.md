# A Short History of Getting Containers Right


You don't need the history to pick a container, but it explains two things you'll run into in engine code: why std::unordered_map is slower than the hash tables outside the standard, and why so many studios ship their own container libraries. The pattern repeats: library authors ship a layout that's faster on real hardware, and the standard adopts it years later. By C++26 the standard has caught up for sorted lookup (flat_map) and for object pools with stable pointers (hive). The hash table is the conspicuous gap: std::unordered_map is still bound by its C++11 interface, while the open-addressing tables outside std report 2-3× better performance[9].
Timeline: from the STL to std::hive

The original containers are still the right default for most code. The rest of this page walks them in order: array, vector, deque, list, map, unordered_map, the adaptors, the views, and the alternatives outside std at the end.

[Back to C++ Containers](index.md)
