# The Decision Cheat Sheet


When you're about to declare a container, the useful question is "what do I need it to do?" This table is organized that way: find the need in the left column.
If you need…	Reach for	Why
A sequence of T, default case 	std::vector<T> 	Contiguous, cache-friendly, O(1) push_back amortized, the right answer for the bulk of sequence workloads[16].
A fixed-size sequence, no heap allocation 	std::array<T, N> 	Size is part of the type. Lives inline in its owner.
A small bounded sequence, usually fits in N, sometimes overflows 	llvm::SmallVector<T, N> or eastl::fixed_vector 	Inline buffer for N, heap fallback past N. Zero allocations on the common path.
O(1) push and pop at both ends 	std::deque<T> 	Chunked storage means front pushes don't shift everything.
O(1) splice between containers, or stable pointers across all mutations 	std::list<T> or std::hive<T> (C++26) 	A pointer to element X survives any operation that doesn't erase X. list is node-based and keeps order; hive keeps pointers stable in contiguous blocks, iterates faster, but doesn't keep order or splice single elements.
Lookup by key, frequent inserts and erases, doesn't fit in a flat_map 	absl::flat_hash_map<K, V> or folly::F14FastMap<K, V> 	Open addressing with SIMD-filtered probing; Google reported 2-3× better performance than std::unordered_map[9]. Rehash moves elements, so pointers into it don't survive growth.
Lookup by key, small to medium static-ish key set, read-heavy 	std::flat_map<K, V> (C++23) or boost::container::flat_map 	Sorted vector. Binary search on contiguous memory is competitive with hashing at small N, and iteration is in key order[4].
Ordered iteration, range queries by key 	std::map<K, V> 	Red-black tree, sorted iteration is automatic, lower_bound/upper_bound work.
LIFO behavior, want to expose only push/pop/top 	std::stack<T, std::vector<T>> 	Vector-backed stack. The deque default works, but a vector keeps the stack in one contiguous block.
FIFO behavior 	std::queue<T> 	The deque default is a good fit. For a bounded queue, especially a lock-free one, use a ring buffer.
"Do the highest-priority job next" 	std::priority_queue<T> 	Vector + binary heap. O(log n) push and pop, O(1) top.
A read-only view of a contiguous range 	std::span<const T> / std::string_view 	Pointer + size. No copy, no allocation, no template-on-container.

[Back to C++ Containers](index.md)
