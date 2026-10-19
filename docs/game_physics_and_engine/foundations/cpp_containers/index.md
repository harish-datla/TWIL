# C++ Containers

An engine walks the same containers thousands of times a frame, and how long that takes often depends more on where the elements sit in memory than on the algorithm: on a desktop x86 core, reading from the L1 cache costs about 4 cycles and a trip to main memory around 250. This page shows what each standard container an engine uses looks like in memory (std::array, std::vector, std::deque, std::list, std::map, std::unordered_map, the adaptors and the views), which of your pointers into it survive which changes, and which one to pick for a given system. Live benchmarks race them in your browser.
Time
~65 min
Level
Mid C++ engineer
Prereqs
You've written a for loop over a vector. You've heard of big-O. The memory-model tutorial is useful if you want to know why unordered_map can't be lock-free.
Hardware
Some idea that DRAM is slower than cache
◂ Build a Game Engine
Phase 0 · Foundations
Next · Hash Tables ▸
What you'll be able to do by the end

    Predict from a container's memory layout roughly how fast it iterates and looks things up, and explain why a vector usually beats a linked list even when both are O(n).
    Explain what "amortized O(1) push_back" promises, and why a single push_back can still cause a hitch.
    Say which operations on each container leave your saved pointers, references and iterators dangling.
    Choose between map, unordered_map, flat_map and an open-addressing table such as SwissTable for a given lookup workload.
    Pass ranges as span and string_view, and plug a per-frame arena into containers with std::pmr.


## Sections

- [Why a Container Choice Is a Performance Decision](01-why-a-container-choice-is-a-performance-decision.md)
- [A Short History of Getting Containers Right](02-a-short-history-of-getting-containers-right.md)
- [The Memory Hierarchy the Container Sits On](03-the-memory-hierarchy-the-container-sits-on.md)
- [std::array: Fixed Size, on the Stack](04-std-array-fixed-size-on-the-stack.md)
- [std::vector: The Workhorse](05-std-vector-the-workhorse.md)
- [Growth, Amortization, and the Cost of push_back](06-growth-amortization-and-the-cost-of-push-back.md)
- [Iterator Invalidation: The Rule You Can't Forget](07-iterator-invalidation-the-rule-you-cant-forget.md)
- [std::deque: Chunked, Two-Ended](08-std-deque-chunked-two-ended.md)
- [std::list and std::forward_list](09-std-list-and-std-forward-list.md)
- [List vs Vector, Measured](10-list-vs-vector-measured.md)
- [std::map and std::set: Ordered Associative](11-std-map-and-std-set-ordered-associative.md)
- [std::unordered_map and the Bucket Interface](12-std-unordered-map-and-the-bucket-interface.md)
- [Open Addressing and SwissTable](13-open-addressing-and-swisstable.md)
- [The Adaptors: stack, queue, priority_queue](14-the-adaptors-stack-queue-priority-queue.md)
- [std::span and std::string_view: Non-Owning Views](15-std-span-and-std-string-view-non-owning-views.md)
- [Beyond std: flat_map, hive, Intrusive Lists, Ring Buffers](16-beyond-std-flat-map-hive-intrusive-lists-ring-buffers.md)
- [Allocators and Why Games Override Them](17-allocators-and-why-games-override-them.md)
- [The Container Race](18-the-container-race.md)
- [The Decision Cheat Sheet](19-the-decision-cheat-sheet.md)
- [The Container Pitfalls List](20-the-container-pitfalls-list.md)
- [What's Next](21-whats-next.md)
