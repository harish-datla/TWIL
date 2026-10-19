# Storage strategy 1: Sparse sets

A buff system adds and removes a Stunned component on hundreds of entities a second, and every add and remove has to be cheap. A sparse set maps entity IDs to component data using two arrays. The sparse array is indexed by entity ID and stores the index into a dense array. The dense array stores entity IDs (and, in parallel, component values) packed contiguously with no gaps.

Worked example

Give Health to entities 7, 2 and 9, in that order. The dense array becomes [7, 2, 9], and the sparse array records sparse[7] = 0, sparse[2] = 1, sparse[9] = 2. Remove entity 7: move the last entry, 9, into slot 0 and pop, so dense is [9, 2] and sparse[9] becomes 0. Checking entity 2 is sparse[2] = 1 and dense[1] = 2, a match, two array reads.

All three core operations are O(1) on the dense side:

    Has: check sparse[entity]. If the value is in range and dense[sparse[entity]] == entity, the entity has this component.
    Add: set sparse[entity] = dense.length, push the entity ID onto dense, push the component value onto the parallel values array.
    Remove: swap the entity's slot with the last element of dense and values, then pop. Update sparse for the swapped entity. O(1), no allocation.

One caveat, visible in the code below: the first insert of a high entity ID has to grow the sparse array out to that index, which is O(max entity ID), not O(1). Amortized over many inserts it disappears, but a single add() for entity 4,000,000 into an empty pool is a four-million-element allocation. Paging the sparse array is the standard fix, and it is what EnTT does[13].

EnTT[12] uses one sparse set per component type. The trade-off: the sparse array is sized to the maximum entity ID, so it can consume significant memory if entity IDs are large. Paging the sparse array (allocating it in fixed-size pages on demand) mitigates this. The swap-and-pop removal does not preserve insertion order, which matters when anything downstream depends on iteration order (sorted rendering, for example). EnTT offers a way to buy back multi-pool iteration cost: an owning group takes ownership of a set of pools and keeps the entities matching the group tightly packed and identically ordered at the front of each one[14]. Iterating a group is then a parallel walk of aligned arrays, which Caini calls perfect SoA: "no jumps, no branches". Two costs come with it. The packing is maintained on component creation and destruction rather than at iteration time, so the win is paid for on the add/remove side. And an array can only be ordered one way, which caps how many groups a component can join: "One component, one group." Nested groups lift that only for groups where "one literally contains the other"; a pair like <A, B> and <A, C> is rejected, since neither extends the other.
Live · Sparse set operations
add first
add random
remove random
reset
dense count
5
sparse size
16
utilization
31%
The sparse array on top, the dense array below, and readouts for dense count, sparse size and utilization. Press add first or add random a few times, then remove random. The sparse array is indexed by entity ID. The dense array is packed: no gaps, no holes. Removal swaps the target with the last element and pops, keeping the dense array contiguous in O(1). The cost is that iteration order changes on every removal. EnTT provides a sort() operation to restore order when needed (useful for render-order-dependent iteration).
Sparse set allocator
C++
Rust

template<typename T>
struct SparseSet {
    static constexpr uint32_t INVALID = UINT32_MAX;

    std::vector<uint32_t> sparse;   // entity ID -> dense index
    std::vector<uint32_t> dense;    // packed entity IDs
    std::vector<T>          values;  // component data, parallel to dense

    void ensure_sparse(uint32_t entityId) {
        if (entityId >= sparse.size())
            sparse.resize(entityId + 1, INVALID);
    }

    bool has(uint32_t entityId) const {
        return entityId < sparse.size()
            && sparse[entityId] != INVALID
            && dense[sparse[entityId]] == entityId;
    }

    void add(uint32_t entityId, T value) {
        ensure_sparse(entityId);
        sparse[entityId] = static_cast<uint32_t>(dense.size());
        dense.push_back(entityId);
        values.push_back(std::move(value));
    }

    void remove(uint32_t entityId) {
        if (!has(entityId)) return;
        auto idx  = sparse[entityId];
        auto last = static_cast<uint32_t>(dense.size() - 1);
        if (idx != last) {                            // swap with last
            dense[idx]  = dense[last];
            values[idx] = std::move(values[last]);
            sparse[dense[idx]] = idx;                // fix swapped entity's sparse entry
        }
        dense.pop_back();
        values.pop_back();
        sparse[entityId] = INVALID;
    }

    T& get(uint32_t entityId)       { return values[sparse[entityId]]; }
    const T& get(uint32_t entityId) const { return values[sparse[entityId]]; }
};

What's intentionally missing

This sparse set is written to be read, not shipped. A production pool adds:

    A guard in add() against adding the same component twice. As written, a second add() pushes a duplicate dense entry and overwrites the sparse slot, orphaning the first one so remove() leaves a stale entity behind. The live widget above guards this; the listing does not.
    Validity checking in the C++ get(). It indexes values[sparse[entityId]] with no check, so calling it for an entity that lacks the component reads out of bounds. The Rust pane returns Option instead, which is the difference worth noticing between the two panes.
    A paged sparse array, so entity ID 4,000,000 does not force a four-million-element allocation[13].
    Pointer stability across insertion, if anything outside the pool holds references into values. Swap-and-pop moves elements, so it does not have it.
    Iteration order guarantees. Swap-and-pop scrambles order (deterministically, given the same operations), which matters when anything depends on processing entities in a particular order.

