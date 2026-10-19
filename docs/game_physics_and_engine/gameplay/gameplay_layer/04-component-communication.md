# Component communication

Components need to talk (a health component tells the UI bar, a trigger tells a door) without every component holding a hard pointer to every other, which would bring back the coupling composition was meant to remove. Three options that coexist[2]: direct references (fast, fine within one entity, dangerous across), the observer pattern, and an event queue.
Immediate vs deferred is a tradeoff, not a winner

The observer (immediate): the subject fires onNotify synchronously and blocks until every observer returns[5]. Simple and debuggable, but watch the lapsed-listener leak (a destroyed observer never unsubscribed becomes a zombie), notification order dependence, and reentrancy (a handler that mutates the subject mid-notify). The event queue (deferred): enqueue and return, drain once per frame[6]. Decoupled and batchable, but adds latency, ordering questions, and stale-world risk (the world changed since you sent it). A subtle one: the queue hides reentrancy by unwinding the stack, so spurious events keep circulating from frame to frame without ever crashing, and the bug goes unnoticed. Neither model is "correct"; real engines use both.

Worked example

A Health component fires "damaged" to three listeners: the HUD bar, the hit-sound player and the AI. Immediate, all three run inside takeDamage() before it returns, so the AI can react in the same tick. Deferred at 60 fps, the event waits in the queue and the three run when it drains, up to 16.7 ms later, and the AI reacts a frame late.

The widget runs the same bus both ways:
immediate
deferred
fire event
reentrant handler
lapsed listener
An emitter, the bus, and its subscribers. Fire an event in each mode, then tick reentrant handler and lapsed listener and fire again. Immediate: the event is a synchronous call chain (watch the stack depth). Deferred: it queues and drains next frame (watch the latency). Turn on reentrant handler and immediate mode grows the call stack while deferred quietly sloshes events across frames (hidden, not fixed). Turn on lapsed listener and a destroyed-but-subscribed node lingers as a zombie still counting events.
An event bus with immediate emit and a deferred queue
C++
Rust

template<class E>
class Channel {
    std::vector<std::function<void(const E&)>> listeners;
    std::vector<E> pending;
public:
    void subscribe(std::function<void(const E&)> fn) { listeners.push_back(std::move(fn)); }
    void emit(const E& e) {                          // IMMEDIATE: synchronous, blocks
        auto snapshot = listeners;                   // copy: a handler that subscribes mid-emit would
                                                     // otherwise invalidate this loop's iterators
        for (auto& fn : snapshot) fn(e);
    }
    void enqueue(E e) { pending.push_back(std::move(e)); }  // DEFERRED
    void drain() {                                   // once per frame
        auto batch = std::move(pending); pending.clear();  // new events wait for next frame
        for (auto& e : batch) emit(e);
    }
};

