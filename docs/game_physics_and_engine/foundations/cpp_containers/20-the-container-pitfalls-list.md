# The Container Pitfalls List


Use this as a review checklist. Each item is a mistake that compiles cleanly and passes most tests, organized by the container that produces it:
vector pitfalls

    vector<bool> is not a container of bool. The standard permits a packed representation (one bit per element) and explicitly carves out the contiguous-storage and ordinary-reference guarantees so implementations can pack; every shipping implementation does. operator[] returns a proxy, &v[0] isn't a bool*, and the type doesn't satisfy the normal sequence-container requirements[19]. Use std::vector<char> or std::vector<std::uint8_t> when you need real bools in contiguous memory, or std::bitset when the size is fixed.
    Iterators held across push_back can dangle. Any push that reallocates invalidates every iterator, pointer and reference, and every push invalidates end(). This is one of the most common invalidation bugs. Hold an index instead, or reserve() enough capacity up front.
    shrink_to_fit() is a non-binding request. It is allowed to do nothing (the major implementations do honor it). To force the issue, swap with a right-sized copy, std::vector<T>(v.begin(), v.end()).swap(v), or with an empty vector, std::vector<T>().swap(v), to release everything.

deque pitfalls

    MSVC's deque collapses to one element per chunk for any element larger than 8 bytes. A deque<MyStruct> on MSVC costs about one heap allocation per element, with the cache misses that implies. libstdc++ and libc++ size chunks in bytes (512 and 4096 respectively), so their chunks hold many elements until elements get large[22].
    Iteration has a per-chunk cost. Each increment checks for the end of the current chunk; within a chunk the branch is predictable, but each chunk transition typically costs a branch mispredict plus a load through the map. For hot inner loops that run many times over the same data, a vector is faster.

list pitfalls

    It is rarely the right answer. Reach for it only when you have a real splice or reference-stability requirement. See §10.
    size() could be O(n) before C++11. C++03 only said it "should" be constant time, and libstdc++'s was linear so that splicing a range stayed O(1). C++11 requires O(1) size(), which makes range splice between two lists O(n) instead. libstdc++ built with its old ABI (_GLIBCXX_USE_CXX11_ABI=0) still has the linear size().

map pitfalls

    m[key] default-constructs the value if the key isn't present and writes the new entry. A "lookup" that mutates the map. Use find or contains (C++20) for "is it there?" checks.
    String-keyed lookups can build a temporary string. With the default comparator, m.find("foo") on a map<std::string, V> constructs a temporary std::string, which allocates when the key is longer than the string's inline (SSO) buffer. Since C++14, declaring the map with std::less<> makes the comparator transparent, and find compares against the const char* directly[29].

unordered_map pitfalls

    The default integer hash is the identity on libstdc++ and libc++. Their prime bucket counts make that work, but code that reuses std::hash in its own power-of-two table (or moves to one) keeps only the low bits, so keys that share their low bits (aligned pointers, IDs in steps of 256) collide. Mix the hash first, or use a hasher that does. MSVC's std::hash for integers already mixes (FNV-1a over the bytes).
    Insertion may rehash, which invalidates all iterators (but not pointers/references). Code that inserts in a loop with iterators outstanding has to call reserve() first, just like vector[20].
    It is usually the slowest hash table available. If a hash table shows up in your profile, swap in absl::flat_hash_map, boost::unordered_flat_map or folly::F14FastMap. The interface is source-compatible enough for most call sites, but the invalidation rules differ: flat storage means a rehash invalidates pointers and references too, not just iterators. Check your reference-holding code before swapping, or use a node-based variant such as absl::node_hash_map.


[Back to C++ Containers](index.md)
