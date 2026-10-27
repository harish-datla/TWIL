# Iterator Invalidation: The Rule You Can't Forget


An AI system saves a pointer to the player's entry in a std::vector of targets. Next frame an enemy spawns, push_back reallocates the vector, and the saved pointer now points into freed memory. Nothing crashes yet: the old bytes are still there, so the AI keeps reading a stale copy of the player until something else reuses that memory.

Every container has a set of operations that invalidate some of the iterators, pointers and references you've saved into it, meaning they no longer point at a live element. Get it wrong and you have a dangling pointer that the compiler can't catch. The rules are precise and worth memorizing for the containers you use most:
Container	What invalidates iterators	What invalidates pointers/references
vector 	Anything that triggers a reallocation invalidates all iterators. Insert or erase invalidates iterators from the change point to end(). 	Same as iterators. References are pointers in disguise here.
deque 	Any insert invalidates all iterators, including push_front and push_back. Erase at the front or back invalidates only the erased iterators (plus the past-the-end iterator when erasing the back); erase in the middle invalidates all. 	Insert at front or back leaves all pointers/references valid; insert or erase in the middle invalidates everything. Erase at the front or back invalidates only the erased element’s references.
list, forward_list 	Only the iterators to erased elements are invalidated. 	Only the pointers to erased elements. Everything else survives.
map, set, multi* 	Same as list: only the erased ones. 	Same as list: only the erased ones.
unordered_map, unordered_set 	Insert may trigger a rehash, which invalidates all iterators. Erase invalidates only the erased iterator. 	Insert with no rehash leaves pointers/references valid; rehash leaves pointers/references valid (only iterators are invalidated). Erase invalidates only the erased element's references.

Adapted from the cppreference invalidation tables[19][20] and the formal ISO wording[3]. The unordered_map rule is the easy one to get wrong: a rehash invalidates iterators but not pointers or references, because the nodes stay where they are and only the bucket links change. That has held since C++11. The table leaves out shrink_to_fit, which may reallocate vector and deque storage and so can invalidate everything.

Worked example

The widget below starts with a vector of a, b, c, capacity 4, and two saved iterators: it₁ at index 0 and it₂ at index 2.

push_back(d): size 4, capacity 4, no reallocation. Both iterators are still valid (only end() moved).

push_back(e): size would be 5 but capacity is 4, so the vector moves to a new block of 8. Both iterators are now dangling.

From the start instead, insert in the middle at index 2: no reallocation, but everything from index 2 shifts right. it₂ is dangling; it₁, before the insertion point, stays valid.

Each button on the widget below performs one operation on a small vector. Two saved iterators point at fixed positions and turn red as soon as an operation invalidates them. A reallocation (including reserve past the current capacity) invalidates both; an insert or erase in the middle invalidates everything from that position onward.
Live · iterator invalidation
push_back ▸
insert middle ▸
erase middle ▸
reserve(2× cap) ▸
Reset ⟲
size
3
capacity
4
it₁ status
valid
it₂ status
valid
The cells are the vector's slots (filled and spare), and the two arrows are saved iterators; the readouts give size, capacity and each iterator's status. Press push_back twice from the start: the second push reallocates and both arrows go red. Reset, then try insert middle and erase middle. A reallocation moves the entire block, so every iterator into the old block becomes a pointer into freed memory. Even an insert that doesn't reallocate (capacity sufficient) shifts every element after the insertion point, so iterators that pointed there now refer to a different element. Using an invalidated iterator is undefined behavior, and tests often miss it because freed memory still holds the old values until something reuses it.

[Back to C++ Containers](index.md)
