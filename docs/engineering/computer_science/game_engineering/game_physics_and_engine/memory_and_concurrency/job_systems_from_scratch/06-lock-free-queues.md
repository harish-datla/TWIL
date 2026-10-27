# Lock-Free Queues


Back to the queue from §3, whose mutex let only one worker in at a time. We have N workers and we want them to push and pop concurrently without a mutex. The phrase to look up is lock-free queue: a queue built from atomic read-modify-write operations such as compare-and-swap (CAS) instead of locks, with the guarantee that a stalled thread can't stop the others from making progress. Done right, multiple workers make progress simultaneously instead of serializing through a critical section. Production examples include moodycamel's ConcurrentQueue[42].

The classic design is Dmitry Vyukov's bounded MPMC queue[16]: a fixed-size array where each slot has a sequence counter, and one CAS per enqueue or dequeue. Vyukov notes it is "not lockfree in the official meaning" (a stalled producer can block consumers waiting on that specific slot), but it is widely used, Rust's crossbeam ArrayQueue among others, because the fast path touches two cache lines and takes no lock.

The enqueue below is the producer's side: it finds the next slot, checks that the slot is free, and claims it with one CAS:
eq. 2 · Vyukov enqueue (sketch) pos ← load(tail) slot ← buf[pos & mask]
if slot.seq = pos: CAS(tail, pos, pos+1); if succeeded, write payload, set seq = pos+1.

In words: each slot tracks its own version (the sequence number). Producers race for the slot whose sequence equals the current tail. Whoever wins the CAS writes the payload and bumps the sequence to mark the slot full; whoever loses gets the new tail back from its failed CAS and tries the next slot. Consumers race symmetrically on the head.

Worked example

A 12-slot ring like the widget's. Every slot starts with its sequence equal to its index. (The sketch's pos & mask needs a power-of-two capacity; with 12 slots it's pos mod 12.)

Producer at tail 5: slot 5's sequence is 5, equal to the tail, so it CASes the tail from 5 to 6, writes the payload, and sets the sequence to 6, "full".

Consumer at head 5: it wants sequence head + 1 = 6, finds it, reads the payload, and sets the sequence to 5 + 12 = 17.

Next lap, the producer at tail 17 lands on slot 17 mod 12 = 5 and finds sequence 17: free. A producer that gets there before the consumer has read finds 6, not 17, and knows the ring is full.

Run the queue below. Producers (purple) push payloads; consumers (green) pull them. Sequence numbers on each slot flip from "ready to write" to "ready to read" and back:
Live · MPMC Queue
producers 2
consumers 3
rate 1.0×
Run ▶
Pause ⏸
Reset ⟲
enqueued
0
dequeued
0
contended CAS
0
The ring's slots show their sequence numbers; purple producers write payloads and green consumers read them. A slot's number steps from "ready to write" to "ready to read" and back as the two sides pass. Producers and consumers never read each other's position counter; they coordinate through each slot's sequence number. A producer's release store to the sequence publishes the payload to the consumer whose acquire load sees it. When several producers go for the same slot in one step, one CAS wins and the rest count as contended and retry.

An MPMC queue is the right structure when you have a true many-to-many relationship. In a job system you mostly don't. The usual pattern:

    One queue (a deque, really: a double-ended queue, pushed and popped at both ends) per worker thread.
    The owning worker is the only producer and almost always the only consumer.
    Other workers occasionally steal from the deque when they run out of their own work.

That structure is a work-stealing deque; the Chase-Lev deque is the standard lock-free implementation, and it's the topic of the next section. The MPMC queue earns its keep in the global submission path (the main thread handing top-level work to the scheduler) where many threads might be producers and the cost of contention is amortized over a coarse-grained event.

[Back to Job Systems from Scratch](index.md)
