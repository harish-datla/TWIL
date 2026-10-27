# Hazard Pointers: Safe Memory Reclamation


Thread A has loaded a node pointer from a lock-free stack and is about to read the node's next field. Thread B pops the same node and frees it, and the allocator hands the memory to the particle system. Thread A reads particle data as if it were a list node. Tagged pointers fix the ABA identity problem on a CAS. They don't solve this lifetime problem: if thread A still holds pointer P when thread B frees the object it points to, thread A's next dereference is undefined behavior, whatever the tag says. Something has to keep P's target alive until thread A is done with it.

Maged Michael's 2002 PODC paper[24] introduced hazard pointers: each thread has a few published pointer slots naming the nodes it's currently using. A thread that unlinks a node doesn't free it directly; it retires it, and from time to time scans every thread's slots and frees only the retired nodes no slot names. Readers publish what they're looking at; reclaimers check before freeing.
hazard_pointers_sketch.cpp · the protocol
Copy

// One hazard slot per thread is enough for the Treiber stack's pop;
// Michael's paper uses two per thread for the Michael-Scott queue.
thread_local std::atomic<Node*> hazardSlot{nullptr};

// Reader side: load the pointer, publish it, then re-check the source.
Node* protectedLoad(std::atomic<Node*>& source) {
  Node* candidate;
  do {
    candidate = source.load(std::memory_order_acquire);
    // Publish what we're about to dereference.
    hazardSlot.store(candidate, std::memory_order_seq_cst);
    // Re-check with seq_cst so the store above can't be reordered after this
    // load (the SB shape from §4). If source still holds candidate, any thread
    // that unlinks it from now on will find our hazard when it scans.
  } while (candidate != source.load(std::memory_order_seq_cst));
  return candidate;
}

// Reclaimer side: call retire() after unlinking a node with a seq_cst CAS.
thread_local std::vector<Node*> retireList;

void retire(Node* node) {
  retireList.push_back(node);
  if (retireList.size() >= 128) scanAndFree();   // amortize the scan over many frees
}

void scanAndFree() {
  // Snapshot every thread's hazard slot. allThreadHazardSlots() is a registry
  // of each thread's slot address, not shown here.
  std::unordered_set<Node*> protectedSet;
  for (std::atomic<Node*>* threadSlot : allThreadHazardSlots())
    if (Node* hazard = threadSlot->load(std::memory_order_seq_cst))
      protectedSet.insert(hazard);

  // Free retired nodes no thread has published; keep the rest for next time.
  for (auto retired = retireList.begin(); retired != retireList.end(); ) {
    if (!protectedSet.contains(*retired)) { delete *retired; retired = retireList.erase(retired); }
    else                                   { ++retired; }
  }
}

Worked example

Eight threads, one hazard slot each, and the listing's threshold of 128 retired nodes per scan.

When a thread's retire list reaches 128 it scans. At most 8 nodes can be named in hazard slots, so the scan frees at least 128 − 8 = 120 nodes and keeps at most 8 for next time.

So no thread ever holds more than 128 retired nodes, and the whole program at most 8 × 128 = 1,024, however long a reader stalls. Each scan does 8 slot loads and 128 set lookups, about one lookup per node freed.

The seq_cst orderings are required. Publishing the hazard and re-checking the source is a store followed by a load of a different location, the SB pattern from §4, and the reclaimer's unlink followed by its scan is the mirror image. With anything weaker, the reader's hazard store can still be sitting in its store buffer when it re-checks the source, while the reclaimer unlinks the node, scans, misses the hazard, and frees a node the reader is about to dereference. Michael's paper requires a full memory barrier between setting a hazard pointer and validating it[24]; in C++ that means seq_cst on those four operations, or a seq_cst fence on each side. C++26 adds std::hazard_pointer[25], and folly's hazptr[26] is a production implementation you can use today.
Hazard pointers vs epoch reclamation

The other production technique is epoch-based reclamation (EBR): a reader announces the current global epoch while it reads; retired nodes go into a bucket for the epoch they were retired in; a bucket is freed once every active reader has moved past that epoch. Reads are cheaper than with hazard pointers (no per-pointer publish and re-check), but a single stalled or preempted reader blocks all reclamation, so unreclaimed memory can grow without bound, which hazard pointers prevent. Linux's RCU (read-copy-update) is a close relative (quiescent-state-based reclamation), and Rust's crossbeam-epoch is a widely used EBR implementation[27].

[Back to The C++ Memory Model from Scratch](index.md)
