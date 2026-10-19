# The ABA Problem


A lock-free free list hands out particle nodes to several threads, and under heavy load it now and then hands the same node to two of them, or loses a run of nodes entirely. No single line of the code looks wrong. A common culprit is a bug called ABA. Lock-free designs lean heavily on compare-and-swap (CAS): "if the atomic still holds the value I read, replace it with this new value." Most multi-producer queues, Treiber stacks, and work-stealing deques are built on it. It has one classic failure mode: ABA.

Thread A reads pointer P and sees value X, then is preempted. Thread B pops X from the structure, frees it, allocates a new node, and the allocator happens to hand back the same address. Thread B pushes the new node, so P holds X again, but it's a different object. Thread A wakes up, does a CAS expecting X, succeeds, and corrupts the structure, because it acts on the recycled address as if it were the original node.

The standard example is the Treiber stack[22], the simplest lock-free LIFO. Its pop:
treiber_stack_aba.cpp · the broken version
Copy

struct Node {
  int value;
  Node* next;
};

std::atomic<Node*> topOfStack{nullptr};

Node* pop() {
  Node* oldTop = topOfStack.load(std::memory_order_acquire);
  while (oldTop) {
    // Read top->next BEFORE the CAS. This is where ABA bites (and, if
    // another thread already freed oldTop, this read is a use-after-free).
    Node* newTop = oldTop->next;
    if (topOfStack.compare_exchange_weak(oldTop, newTop,
          std::memory_order_acq_rel,
          std::memory_order_acquire)) {
      return oldTop;
    }
    // CAS failed; oldTop now holds the current top. Try again.
  }
  return nullptr;
}

// The race:
// Thread A: reads top = X (a Node with X->next = Y). About to CAS X -> Y.
// Thread B: pop X (top is now Y). pop Y (top is now Z). push X (recycled!).
//           Now top = X, but X->next = Z (B set it when pushing).
// Thread A: CAS expects X, finds X, succeeds. Sets top = Y. But Y was freed!
// Result: top points at freed memory, or the stack has lost nodes Z, ...

Three families of fix:

    Tagged pointers. Pack a counter into bits of the pointer that don't hold address[23]. With 4-level paging, x86-64 user addresses fit in 48 bits, leaving the top 16 free; 5-level paging (Ice Lake server parts and later) widens addresses to 57 bits and leaves 7. AArch64 usually uses 48-bit addresses, and its Top Byte Ignore feature lets the top 8 bits carry a tag without masking, though pointer authentication (PAC) claims some high bits when enabled. The CAS compares pointer and counter together; each successful pop or push bumps the counter, so a recycled pointer comes back with a different tag and a stale CAS fails.
    Double-width CAS. Use std::atomic<TaggedPtr> where TaggedPtr is a 16-byte struct (pointer plus 64-bit counter). Hardware has the instruction (CMPXCHG16B on x86-64, CASP on ARMv8.1+), but whether the toolchain uses it inline varies: Clang does with -mcx16, while GCC routes 16-byte atomics through calls into libatomic. Check is_always_lock_free before relying on it.
    Memory reclamation. Don't free a popped node until no other thread can still be reading it. Hazard pointers (§14) and epoch-based reclamation are the production techniques, and they solve ABA as a side effect: an address can't be reused while someone still holds it.

Worked example

The tagged version of the widget's race. Thread 1 reads top = (X, tag 1) and X→next = Y, then is preempted. Thread 2 pops X (tag 2), pops Y (tag 3) and pushes a new node that reuses X's address (tag 4).

Thread 1 wakes and runs its CAS expecting (X, 1). The top is (X, 4): same pointer, different tag, so the CAS fails, and thread 1 retries with top = X and next = Z.

How many bits the tag gets decides how safe that is. With 48-bit addresses there are 16 spare bits, so the tag repeats after 216 = 65,536 updates; with 5-level paging only 7 bits are left, and it repeats after 128. ABA comes back if thread 1 sleeps through exactly a multiple of that many updates and the address is reused. A 64-bit counter in a double-width CAS makes that practically impossible.
Live · ABA stack visualizer
no tag
tagged pointer
Play race ▶
Step ▸
Reset ⟲
step
0
CAS result
···
stack state
top=X
The stack is drawn top first, with what each thread holds beside it and the tag on the top pointer. Press Step to walk the race one event at a time, or Play; then switch to tagged pointer and run it again. Without a tag, thread 1's CAS succeeds and leaves the top pointing at the freed node Y. With a tag, the CAS compares pointer and tag together, sees that the tag changed, fails, and retries with fresh values. Tagged pointers fix ABA on the CAS itself; they don't make it safe to free nodes that another thread may still dereference. That needs hazard pointers or epoch-based reclamation.

[Back to The C++ Memory Model from Scratch](index.md)
