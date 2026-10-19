# Actions and conditions

Everything so far decides which leaf runs. The leaves are where the guard actually checks the world or does something, and the hard case is an action that takes longer than one frame, such as walking across a room.

Leaf nodes are where the actual work happens. Two kinds:

    Conditions test world state and return SUCCESS or FAILURE in a single tick. IsTargetVisible, IsHealthBelow(30), HasAmmo, IsInCover. They never return RUNNING.
    Actions perform work. MoveTo, Attack, PlayAnimation, Reload, Wait(t). Actions that complete in one tick return SUCCESS or FAILURE. Actions that span multiple frames (movement, animation playback) return RUNNING until they finish.

The RUNNING status is what makes BTs handle multi-frame actions cleanly. A MoveTo action that takes 200 frames to complete returns RUNNING on every frame until the last. The parent sequence sees RUNNING, returns RUNNING itself, and the tree pauses evaluation of that branch until next tick. This propagation continues all the way up to the root.

Worked example

A guard walks 10 m to cover at 3 m/s. That takes 10 / 3 ≈ 3.33 s, which is 200 frames at 60 fps. MoveTo returns RUNNING on frames 1 to 199 and SUCCESS on frame 200. On each of those 199 frames, its sequence and every composite above it return RUNNING too, so the root reports RUNNING and the game loop carries on rendering the walk.

The composites below are reactive: every tick starts again at child 0, which is the default Colledanchise and Ögren build on[3] and the baseline §11 compares memory against. Each one still records which child was RUNNING, for one reason: when an earlier child ends the tick first (a condition fails, a higher-priority branch takes over), the child that was running has to be halted, or it keeps a half-finished action alive.
Code: base node, sequence, selector
Node hierarchy + composites
C++
Rust

#include <memory>
#include <vector>

enum class Status { Success, Failure, Running };

// Base node. Every node implements tick(); halt() is optional.
class Node {
public:
    virtual ~Node() = default;
    virtual Status tick() = 0;

    // Called when a parent abandons this node while it was RUNNING.
    // Anything the node started and has not finished gets undone here.
    virtual void halt() {}
};

// Sequence: AND logic. Fail on first failure, succeed when all succeed.
// Reactive: every tick starts again at child 0.
class Sequence : public Node {
    std::vector<std::unique_ptr<Node>> children;
    int runningIndex = -1;  // child that returned RUNNING last tick, or -1
public:
    void addChild(std::unique_ptr<Node> child) {
        children.push_back(std::move(child));
    }

    // Abort is recursive: a composite that is dropped must drop whatever
    // it had running underneath, or that work leaks.
    void halt() override {
        for (auto& child : children) child->halt();
        runningIndex = -1;
    }

    Status tick() override {
        const int count = static_cast<int>(children.size());
        for (int i = 0; i < count; ++i) {
            Status result = children[i]->tick();
            if (result == Status::Success) continue;
            // An earlier child ended this tick before the one that was running
            // got ticked. Halt it, or its half-finished action lives on.
            if (runningIndex > i) children[runningIndex]->halt();
            runningIndex = (result == Status::Running) ? i : -1;
            return result;             // FAILURE or RUNNING propagates up
        }
        runningIndex = -1;
        return Status::Success;        // all children succeeded
    }
};

// Selector: OR logic. Succeed on first success, fail when all fail.
// Reactive: higher-priority children are re-checked every tick.
class Selector : public Node {
    std::vector<std::unique_ptr<Node>> children;
    int runningIndex = -1;  // child that returned RUNNING last tick, or -1
public:
    void addChild(std::unique_ptr<Node> child) {
        children.push_back(std::move(child));
    }

    void halt() override {
        for (auto& child : children) child->halt();
        runningIndex = -1;
    }

    Status tick() override {
        const int count = static_cast<int>(children.size());
        for (int i = 0; i < count; ++i) {
            Status result = children[i]->tick();
            if (result == Status::Failure) continue;
            // A higher-priority child took over from the one that was running.
            // That switch is the abort, and the loser must be halted.
            if (runningIndex > i) children[runningIndex]->halt();
            runningIndex = (result == Status::Running) ? i : -1;
            return result;             // SUCCESS or RUNNING propagates up
        }
        runningIndex = -1;
        return Status::Failure;        // all children failed
    }
};

Code: a concrete action (MoveTo with RUNNING)
Multi-frame action node
C++
Rust

// Restated from the neighbouring panes so this one compiles on its own.
enum class Status { Success, Failure, Running };
struct Vec3 { float x, y, z; };
class Node { public: virtual ~Node() = default; virtual Status tick() = 0; virtual void halt() {} };
struct Blackboard { template <typename T> T get(const char*) const { return T{}; } };
struct NavAgent { bool requestPath(Vec3); bool hasArrived() const; void cancelPath(); };

// MoveTo: navigate to a target position read from the blackboard.
// Returns RUNNING while pathfinding/moving, SUCCESS on arrival,
// FAILURE if the path is invalid or the target is unreachable.
class MoveTo : public Node {
    Blackboard& blackboard;
    NavAgent&   navAgent;
    bool started = false;
public:
    MoveTo(Blackboard& blackboard, NavAgent& navAgent)
        : blackboard(blackboard), navAgent(navAgent) {}

    Status tick() override {
        if (!started) {
            Vec3 target = blackboard.get<Vec3>("target.position");
            if (!navAgent.requestPath(target))
                return Status::Failure;  // no valid path
            started = true;
        }
        if (navAgent.hasArrived()) {
            started = false;          // reset for next invocation
            return Status::Success;
        }
        return Status::Running;      // still moving
    }

    // Without this, a higher-priority branch that preempts us mid-move
    // leaves started == true. On re-entry we skip requestPath and poll
    // hasArrived() against a stale path. That is the abort bug §11 is about,
    // and it is invisible until a designer adds a higher-priority branch.
    void halt() override {
        if (started) {
            navAgent.cancelPath();
            started = false;
        }
    }
};

Code: blackboard with typed access
Blackboard key-value store
C++
Rust

#include <any>
#include <string>
#include <unordered_map>

// Blackboard: type-erased key-value store.
// Production code would use a fixed-layout struct for cache locality;
// the std::any version here prioritizes flexibility for prototyping.
class Blackboard {
    std::unordered_map<std::string, std::any> data;
public:
    template<typename T>
    void set(const std::string& key, T value) {
        data[key] = std::move(value);
    }

    template<typename T>
    T get(const std::string& key) const {
        auto it = data.find(key);
        if (it == data.end())
            return T{};               // default-constructed if missing
        return std::any_cast<T>(it->second);  // throws std::bad_any_cast on a type mismatch
    }

    bool has(const std::string& key) const {
        return data.count(key) > 0;
    }
};

What's intentionally missing

The code above skips: memory composites, Parallel and decorators (§7, §8, §11); thread safety (a blackboard read and written from a job system needs atomics or locks); key-change notifications (the observers behind Unreal's event-driven aborts); typed key definitions (catching "target.positon" typos at compile time); and scope hierarchies (per-entity vs per-squad vs global). A shipping blackboard also usually resolves string keys to integer IDs once, at load time, so a lookup is an array index rather than a string hash and compare.
