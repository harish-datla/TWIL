# std::unordered_map and the Bucket Interface


An asset manager looks up loaded textures by name hundreds of times a frame and never needs them in order. That's a job for a hash table: turn the key into a number with a hash function, use the number to pick a bucket, and look only in that bucket. std::unordered_map is the standard one. It gives average constant-time lookup, insert and erase, and worst-case linear time when many keys collide in one bucket. C++11 standardized it with an implementation strategy baked into the interface: closed addressing (each bucket holds a linked list of entries that hashed to it), exposed through the public bucket_count(), bucket_size(), and local_iterator APIs[20]. Together with the pointer-stability guarantee, that interface locks every shipping std::unordered_map into separate chaining[7], which is a large part of why open-addressing tables outside the standard beat it, by 2-3× in Google's measurements[9].

The widget shows the closed-addressing layout. Each bucket points to a linked list of {hash, key, value} nodes. Lookup hashes the key, picks the bucket, then walks the bucket's list comparing keys. The load factor (size() / bucket_count()) controls how long the bucket chains get; the default max_load_factor is 1.0, so the average chain holds at most one element.
Live · unordered_map (separate chaining)
key to insert 42
load-factor target 1.00
insert ▸
find ▸
seed 20 random ▸
Reset ⟲
size
20
buckets
32
load factor
0.63
max chain
3
Each numbered slot is a bucket, with its chain of nodes hanging below it; the readouts give size, bucket count, load factor and the longest chain. The table starts with about 20 random keys. Pick a key and press insert or find to highlight its bucket and node, or lower the load-factor target and seed more keys to watch it rehash into twice as many buckets. Separate chaining: each bucket is the head of a linked list. A lookup can miss the cache once reading the bucket and again for each chain node it visits. The interface of std::unordered_map bakes this in: the bucket API with its local_iterator, node-based pointer stability, and user-controlled load factor are requirements an open-addressing table can't meet[7]. The widget doubles the bucket count on rehash; libstdc++ and libc++ move to a prime near double.
Hash quality matters more than you think

The hash function decides whether the table runs at O(1) or degenerates to O(n). std::hash<int> on libstdc++ and libc++ is the identity function: hash(42) == 42. That works because both libraries use prime bucket counts and take hash % bucket_count, which mixes every bit of the key into the bucket index. The same identity hash fails badly in a table with a power-of-two bucket count, which keeps only the low bits: keys that share their low bits (16-byte-aligned pointers, IDs that are multiples of 256) all land in the same few buckets. That's why power-of-two tables scramble the hash first.

Worked example

Entity IDs 256, 512, 768 and 1,024, with the identity hash.

8 buckets (a power of two): 256 % 8, 512 % 8, 768 % 8 and 1,024 % 8 are all 0. Every key lands in bucket 0, and lookups walk one long chain.

13 buckets (a prime): the remainders are 9, 5, 1 and 10. Four different buckets.

Hash flooding, where an attacker picks keys that all collide, is a real attack, and the standard library does not defend against it: its hashes are unseeded. Code that hashes untrusted input (network packets, user-supplied strings) should use a keyed hash such as SipHash with a per-process random key[25].

For game code, where the inputs are usually engine-controlled, the default is usually fine. The bigger issue is the table's layout.

[Back to C++ Containers](index.md)
