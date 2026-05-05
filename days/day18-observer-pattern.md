# Day 18: Observer Pattern

[← Back to Study Plan](../lld-study-plan.md) | [← Day 17](day17-builder-pattern.md)

> **Time**: ~1.5-2 hours
> **Goal**: Understand the Observer pattern — how subjects notify subscribed observers without depending on their concrete types, why subscription **lifetime** is the hard part (dangling observers, dangling subjects, reentrancy), and how modern C++ solves it with `std::function` callbacks, subscription tokens, and `weak_ptr`. Build an `EventEmitter` with `on(event, callback) → token`, `emit(event, data)`, `off(event, id)` that's safe against unsubscription **during** emit.

---
---

# PART 1: WHAT IS THE OBSERVER PATTERN AND WHY DOES IT EXIST?

---
---

<br>

<h2 style="color: #2980B9;">📘 18.1 The Core Idea</h2>

A **subject** holds state. **Observers** want to know when that state changes. Observer establishes a one-to-many relationship where the subject **publishes** changes and registered observers **react** — without the subject knowing what kind of objects observe it.

```
Subject (publisher)               Observers (subscribers)
   ┌────────┐                      ┌──────────┐
   │ Stock  │ ──── notify ───────► │ ChartUI  │
   │  Tick  │                      └──────────┘
   │        │                      ┌──────────┐
   │        │ ──── notify ───────► │ Logger   │
   │        │                      └──────────┘
   │        │                      ┌──────────┐
   │        │ ──── notify ───────► │ Alerter  │
   └────────┘                      └──────────┘

  Subject knows nothing about ChartUI/Logger/Alerter.
  It only knows "I have a list of callbacks to invoke."
```

<br>

#### When to reach for it

| Use case | Why Observer fits |
|----------|-------------------|
| **GUI event handling** | Buttons fire `onClick`; multiple listeners react |
| **Model-view separation** | Model changes; views refresh |
| **In-process pub/sub** | Decouple producers from consumers |
| **Property change notifications** | Settings change; cached views recompute |
| **Reactive systems** | Streams of events flowing through transformations |

The whole point: the subject doesn't `#include` observer types, doesn't construct them, doesn't own them. It only owns a list of opaque callbacks.

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 18.2 The Coupling Problem Without Observer</h2>

```cpp
class Stock {
    ChartUI& m_chart;
    Logger&  m_logger;
    Alerter& m_alerter;
public:
    void onTick(double price) {
        m_chart.update(price);
        m_logger.log(price);
        m_alerter.check(price);
    }
};
```

`Stock` is now coupled to `ChartUI`, `Logger`, `Alerter`. Adding a fourth subscriber means editing `Stock`. Removing one means editing `Stock`. The class fans out to every consumer that exists today and every one that will ever exist.

The Observer pattern flips this: the subject keeps a list of subscribers and exposes `subscribe()` / `unsubscribe()`. Consumers come and go without touching the subject.

<br><br>

---
---

# PART 2: IMPLEMENTATION VARIANTS

---
---

<br>

<h2 style="color: #2980B9;">📘 18.3 Variant 1: Classic OOP (Observer Interface)</h2>

The GoF formulation: an `Observer` abstract base class with a single virtual method.

```cpp
class IObserver {
public:
    virtual ~IObserver() = default;
    virtual void onTick(double price) = 0;
};

class Stock {
public:
    void subscribe(IObserver* obs)   { m_observers.push_back(obs); }
    void unsubscribe(IObserver* obs) {
        m_observers.erase(std::remove(m_observers.begin(), m_observers.end(), obs),
                          m_observers.end());
    }

    void onTick(double price) {
        for (auto* o : m_observers) o->onTick(price);
    }

private:
    std::vector<IObserver*> m_observers;
};

class ChartUI : public IObserver {
public:
    void onTick(double price) override { /* redraw */ }
};
```

<br>

#### Trade-offs

| Pros | Cons |
|------|------|
| Type-safe — observers must implement the interface | One class can only subscribe one way (single `onTick`) |
| Easy to factor common behavior into a base class | Subscriber must inherit — no flexibility for free functions or lambdas |
| Works with virtual dispatch | Raw pointers → who owns the observer? |

This is the textbook version, but in modern C++ we usually prefer the callback variant below.

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 18.4 Variant 2: Function-Callback (Modern C++)</h2>

Replace the interface with `std::function`. Anything callable subscribes — lambdas, member function bindings, free functions.

```cpp
using TickCallback = std::function<void(double)>;
using SubscriptionId = std::uint64_t;

class Stock {
public:
    SubscriptionId subscribe(TickCallback cb) {
        SubscriptionId id = m_nextId++;
        m_callbacks.emplace(id, std::move(cb));
        return id;
    }

    void unsubscribe(SubscriptionId id) {
        m_callbacks.erase(id);
    }

    void onTick(double price) {
        for (const auto& [id, cb] : m_callbacks) cb(price);
    }

private:
    SubscriptionId m_nextId = 1;
    std::unordered_map<SubscriptionId, TickCallback> m_callbacks;
};

// Usage:
Stock s;
auto id1 = s.subscribe([](double p) { std::cout << "chart: " << p << "\n"; });
auto id2 = s.subscribe([](double p) { std::cout << "log: "   << p << "\n"; });
s.onTick(101.5);
s.unsubscribe(id1);
```

<br>

#### Why an ID instead of a pointer?

- Callbacks aren't comparable (`std::function` lacks `==`)
- IDs are stable handles — pass them around, store them, drop them
- Unsubscribe is O(1) with a `unordered_map` keyed on ID

This is the dominant style in modern C++ event systems (Qt's signals/slots, Node-style emitters, GUI libraries).

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 18.5 Variant 3: Multi-Event Emitter (the Build Target)</h2>

A single subject often publishes **many distinct events**. Instead of one `Stock` per event type, group them under one `EventEmitter` keyed by event name.

```cpp
class EventEmitter {
public:
    using Callback = std::function<void(const std::any&)>;

    SubscriptionId on(const std::string& event, Callback cb);
    void off(const std::string& event, SubscriptionId id);
    void emit(const std::string& event, const std::any& data);

private:
    /* internal storage — see Build Exercise */
};

// Usage:
EventEmitter bus;
auto id = bus.on("user.login", [](const std::any& d) {
    auto user = std::any_cast<std::string>(d);
    std::cout << user << " logged in\n";
});
bus.emit("user.login", std::string{"alice"});
bus.off("user.login", id);
```

This is the Node.js / DOM `addEventListener` model. We'll build it in PART 6.

<br><br>

---
---

# PART 3: SUBSCRIPTION LIFETIME — THE HARD PROBLEM

---
---

<br>

<h2 style="color: #2980B9;">📘 18.6 The Three Failure Modes</h2>

Observer is conceptually simple. Lifetime is where bugs live.

<br>

#### Failure 1: Observer dies before unsubscribing

```cpp
{
    Stock s;
    Logger logger;
    s.subscribe([&logger](double p) { logger.log(p); });
    // logger goes out of scope HERE
}
// later... s.onTick(100);  → callback captures dangling reference → UB
```

The lambda captured `logger` by reference. When `logger` died, every callback referring to it became a landmine. The subject never noticed.

<br>

#### Failure 2: Subject dies before observer

```cpp
SubscriptionId id;
{
    Stock s;
    id = s.subscribe([](double p) { /* ... */ });
}  // s is destroyed
s.unsubscribe(id);  // accessing destroyed subject → UB
```

The observer holds a token but no way to know whether the subject is still alive.

<br>

#### Failure 3: Reentrant emit

```cpp
class Stock {
    void onTick(double p) {
        for (const auto& [id, cb] : m_callbacks) cb(p);   // ← iterating
    }
};

s.subscribe([&s](double p) {
    s.unsubscribe(some_other_id);   // mutates m_callbacks DURING iteration → UB
});
```

A callback that subscribes/unsubscribes during emit invalidates iterators. Even worse: a callback that emits another event re-enters `emit`.

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 18.7 Solutions to Lifetime Problems</h2>

<br>

#### Solution A: Subscription token with RAII

The act of subscribing returns an opaque RAII handle. Destroying the handle unsubscribes automatically.

```cpp
class Subscription {
public:
    Subscription() = default;
    Subscription(Stock* s, SubscriptionId id) : m_stock(s), m_id(id) {}
    ~Subscription() { if (m_stock) m_stock->unsubscribe(m_id); }

    Subscription(Subscription&& other) noexcept
        : m_stock(std::exchange(other.m_stock, nullptr)),
          m_id(std::exchange(other.m_id, 0)) {}
    Subscription& operator=(Subscription&& other) noexcept {
        if (this != &other) {
            if (m_stock) m_stock->unsubscribe(m_id);
            m_stock = std::exchange(other.m_stock, nullptr);
            m_id    = std::exchange(other.m_id, 0);
        }
        return *this;
    }

    Subscription(const Subscription&) = delete;
    Subscription& operator=(const Subscription&) = delete;

private:
    Stock* m_stock = nullptr;
    SubscriptionId m_id = 0;
};

// Usage:
{
    auto sub = stock.subscribe(...);   // returns Subscription
    // ...
}   // sub destructor unsubscribes — no manual off() needed
```

This solves the **observer-dies-without-unsubscribing** problem: the Subscription handle's lifetime is bound to whoever owns it.

<br>

#### Solution B: `weak_ptr` to the observer

If the observer is a `shared_ptr`-managed object, store a `weak_ptr` in the subject. Before invoking, `lock()` it; if dead, skip.

```cpp
class Stock {
    std::vector<std::weak_ptr<IObserver>> m_observers;
public:
    void subscribe(std::shared_ptr<IObserver> obs) {
        m_observers.push_back(obs);
    }

    void onTick(double p) {
        for (auto it = m_observers.begin(); it != m_observers.end(); ) {
            if (auto strong = it->lock()) {
                strong->onTick(p);
                ++it;
            } else {
                it = m_observers.erase(it);   // observer is dead, prune
            }
        }
    }
};
```

`weak_ptr` is the cleanest solution **when observers are already `shared_ptr`-owned**. Don't force `shared_ptr` ownership just to satisfy the subject.

<br>

#### Solution C: Snapshot the listener list before emit

Reentrancy and during-emit unsubscribe are solved by **iterating a copy**:

```cpp
void emit(const std::string& event, const std::any& data) {
    std::vector<std::pair<SubscriptionId, Callback>> snapshot;
    {
        std::lock_guard lock(m_mutex);
        auto it = m_listeners.find(event);
        if (it == m_listeners.end()) return;
        snapshot.assign(it->second.begin(), it->second.end());
    }
    // Iterate the SNAPSHOT — listeners can on()/off() freely on the live map
    for (const auto& [id, cb] : snapshot) cb(data);
}
```

A subscriber that adds another subscriber during emit will be invoked on the **next** emit (not the current one). A subscriber that removes another will skip it on the next emit. This is the standard semantic — most callers expect it.

<br>

#### Picking a strategy

| Need | Use |
|------|-----|
| Modern C++ default | Subscription tokens (RAII) + snapshot-on-emit |
| Observers are `shared_ptr`-managed | `weak_ptr` storage |
| Async/cross-thread emission | Snapshot + post to executor |
| Tiny embedded / no allocation | Intrusive linked list of observers |

<br><br>

---
---

# PART 4: THREAD SAFETY

---
---

<br>

<h2 style="color: #2980B9;">📘 18.8 Concurrent Subscription and Emit</h2>

Two threads calling `on()` and `emit()` concurrently must not corrupt the listener map. The standard pattern: protect mutations with a mutex; protect emit by snapshotting the list.

```cpp
class EventEmitter {
    std::mutex m_mutex;
    std::unordered_map<std::string, std::vector<Listener>> m_listeners;

public:
    void on(...) {
        std::lock_guard lock(m_mutex);
        // mutate m_listeners
    }

    void emit(const std::string& event, const std::any& data) {
        std::vector<Listener> snapshot;
        {
            std::lock_guard lock(m_mutex);
            auto it = m_listeners.find(event);
            if (it != m_listeners.end()) snapshot = it->second;
        }
        // Invoke callbacks WITHOUT the lock held
        for (const auto& l : snapshot) l.cb(data);
    }
};
```

<br>

#### Why release the lock before invoking callbacks?

Holding the lock during callback invocation is a deadlock factory:
- Callback may call `on()` / `off()` on the same emitter → recursive lock attempt
- Callback may take other locks while holding ours → lock-order inversion
- Callback may run for milliseconds → blocks all other subscribers

**Rule**: emit, snapshot under lock; **invoke unlocked**.

<br>

#### What about callback invocation order across threads?

Once you snapshot, two concurrent emits can deliver in different orders. If observers care about order, you need an additional serialization layer (a single dispatch thread, or per-listener queues). For most pub/sub, "eventually delivered, order within a single emit preserved" is the right contract.

<br><br>

---
---

# PART 5: ANTI-PATTERNS & TRADE-OFFS

---
---

<br>

<h2 style="color: #2980B9;">📘 18.9 Common Mistakes</h2>

<br>

#### Anti-pattern 1: Holding callbacks longer than expected

A `std::function` keeps captured state alive. If a UI button captures `[this]` and the UI is destroyed but the subject isn't, the dangling capture sits there until someone unsubscribes. Use RAII tokens or `weak_ptr` to make this visible.

<br>

#### Anti-pattern 2: Synchronous emit on a hot path

```cpp
void onTick(double p) {
    for (const auto& cb : m_callbacks) cb(p);   // blocks until ALL callbacks return
}
```

If one observer is slow, every other observer waits. For high-frequency emit, push work onto an executor:

```cpp
for (const auto& cb : snapshot) {
    m_pool.post([cb, p] { cb(p); });   // fire-and-forget
}
```

But now you have to think about ordering, backpressure, and observer lifetime under async dispatch. Don't async-dispatch by default; reach for it when measurements demand.

<br>

#### Anti-pattern 3: `IObserver` with twelve methods

```cpp
class IObserver {
    virtual void onTick(double) = 0;
    virtual void onConnect()    = 0;
    virtual void onDisconnect() = 0;
    virtual void onError(...)   = 0;
    /* ... */
};
```

Every observer must implement all twelve. Half of them just do `{}`. Either split into separate event channels (`EventEmitter`) or keep it small.

<br>

#### Anti-pattern 4: Tight observer/subject coupling resurfacing

If your `Logger` observer needs to call back into `Stock` to query state mid-emit, you've recreated coupling. The pattern works best when notifications carry **all** the data the observer needs.

<br><br>

---
---

# PART 6: BUILD EXERCISE

---
---

<br>

<h2 style="color: #2980B9;">📘 18.10 Exercise: <code>EventEmitter</code> with <code>on / emit / off</code></h2>

Build a multi-event emitter that's safe against unsubscribe-during-emit and supports any payload via `std::any`.

<br>

#### Skeleton

```cpp
// event_emitter.h
#pragma once
#include <any>
#include <cstdint>
#include <functional>
#include <mutex>
#include <string>
#include <unordered_map>
#include <vector>

class EventEmitter {
public:
    using SubscriptionId = std::uint64_t;
    using Callback = std::function<void(const std::any&)>;

    SubscriptionId on(const std::string& event, Callback cb) {
        std::lock_guard lock(m_mutex);
        SubscriptionId id = ++m_nextId;
        m_listeners[event].push_back({id, std::move(cb)});
        return id;
    }

    void off(const std::string& event, SubscriptionId id) {
        std::lock_guard lock(m_mutex);
        auto it = m_listeners.find(event);
        if (it == m_listeners.end()) return;
        auto& vec = it->second;
        vec.erase(std::remove_if(vec.begin(), vec.end(),
                                 [id](const Listener& l) { return l.id == id; }),
                  vec.end());
        if (vec.empty()) m_listeners.erase(it);
    }

    void emit(const std::string& event, const std::any& data) {
        std::vector<Listener> snapshot;
        {
            std::lock_guard lock(m_mutex);
            auto it = m_listeners.find(event);
            if (it == m_listeners.end()) return;
            snapshot = it->second;        // copy under lock
        }
        for (const auto& l : snapshot) l.cb(data);   // invoke unlocked
    }

    std::size_t listenerCount(const std::string& event) const {
        std::lock_guard lock(m_mutex);
        auto it = m_listeners.find(event);
        return it == m_listeners.end() ? 0 : it->second.size();
    }

private:
    struct Listener {
        SubscriptionId id;
        Callback cb;
    };
    mutable std::mutex m_mutex;
    SubscriptionId m_nextId = 0;
    std::unordered_map<std::string, std::vector<Listener>> m_listeners;
};
```

<br>

#### Test driver

```cpp
// main.cpp
#include "event_emitter.h"
#include <cassert>
#include <iostream>
#include <string>
#include <thread>
#include <vector>
#include <atomic>

int main() {
    EventEmitter bus;

    // 1. Basic on / emit
    int loginCount = 0;
    auto id1 = bus.on("user.login", [&](const std::any& d) {
        auto name = std::any_cast<std::string>(d);
        std::cout << name << " logged in\n";
        ++loginCount;
    });
    bus.emit("user.login", std::string{"alice"});
    bus.emit("user.login", std::string{"bob"});
    assert(loginCount == 2);

    // 2. Multiple listeners on same event
    int chartCount = 0, logCount = 0;
    bus.on("price.tick", [&](const std::any& d) { ++chartCount; (void)d; });
    bus.on("price.tick", [&](const std::any& d) { ++logCount;   (void)d; });
    bus.emit("price.tick", 101.5);
    bus.emit("price.tick", 102.0);
    assert(chartCount == 2 && logCount == 2);

    // 3. off — second emit doesn't reach the unsubscribed listener
    bus.off("user.login", id1);
    bus.emit("user.login", std::string{"charlie"});
    assert(loginCount == 2);   // still 2

    // 4. Unsubscribe-during-emit safety
    EventEmitter bus2;
    auto idA = bus2.on("evt", [&](const std::any&) {
        std::cout << "A fires\n";
    });
    EventEmitter::SubscriptionId idB = 0;
    idB = bus2.on("evt", [&](const std::any&) {
        std::cout << "B fires; B unsubscribes during emit\n";
        bus2.off("evt", idB);
    });
    bus2.on("evt", [&](const std::any&) {
        std::cout << "C fires (still in snapshot for THIS emit)\n";
    });

    bus2.emit("evt", 0);
    std::cout << "second emit:\n";
    bus2.emit("evt", 0);   // B is gone now
    (void)idA;

    // 5. Concurrent on/emit/off
    EventEmitter bus3;
    std::atomic<int> hits{0};
    std::vector<std::thread> threads;
    for (int i = 0; i < 4; ++i) {
        threads.emplace_back([&] {
            for (int j = 0; j < 1000; ++j) {
                auto id = bus3.on("x", [&](const std::any&) { ++hits; });
                bus3.emit("x", 0);
                bus3.off("x", id);
            }
        });
    }
    for (auto& t : threads) t.join();
    std::cout << "concurrent test hits = " << hits.load() << " (no crash = pass)\n";

    return 0;
}
```

<br>

#### Compile and run

```bash
g++ -std=c++17 -Wall -Wextra -Wpedantic -pthread \
    -fsanitize=thread,undefined \
    -o day18 main.cpp && ./day18
```

<br>

#### Expected output pattern

```
alice logged in
bob logged in
A fires
B fires; B unsubscribes during emit
C fires (still in snapshot for THIS emit)
second emit:
A fires
C fires (still in snapshot for THIS emit)
concurrent test hits = ...  (no crash = pass)
```

<br>

#### Bonus Challenges

1. **RAII subscription token** — change `on()` to return a `Subscription` object whose destructor calls `off()`. Eliminate the manual `off(event, id)` API for typical callers.

2. **Typed events** — replace `std::any` with a typed template: `template<typename T> on<T>(event, std::function<void(const T&)>)`. Use `std::type_index` keys to enforce type safety at runtime.

3. **`once()`** — add `once(event, callback)` that fires the callback at most once, then auto-unsubscribes.

4. **Wildcard subscriptions** — `on("*", callback)` receives every event. Implement as a separate listener list iterated after the named one.

5. **Async emit** — add `emitAsync(event, data)` that posts each callback invocation to a thread pool and returns a `std::future<void>` that completes when all callbacks have finished.

6. **`weak_ptr` observer variant** — implement a sibling class `WeakEventEmitter` whose listeners are `weak_ptr<Listener>`-based. Show that destroying an observer makes its listener silently disappear on next emit.

<br><br>

---
---

# PART 7: DEEP DIVE — COMMON QUESTIONS

---
---

<br>

<h2 style="color: #2980B9;">📘 18.11 Q&A</h2>

<br>

#### Q1: "Why not just use `std::vector<std::function>` and call them all?"

That's exactly what we built — but the wrappers buy you:
- **Identification** (subscription IDs for `off`)
- **Multi-event keying** (one emitter, many event types)
- **Lifetime safety** (snapshot-on-emit)
- **Thread safety** (mutex)

Without those, you have a callback list, not an emitter.

<br>

#### Q2: "How do `Boost.Signals2` / Qt signals/slots compare?"

**Boost.Signals2** offers thread-safe signals with automatic disconnection via `boost::signals2::scoped_connection` (similar to our RAII token). Heavier dependency than `std::function`-based emitters.

**Qt signals/slots** integrate with Qt's meta-object system; emission can cross thread boundaries via the event loop. They use `connect`/`disconnect` and rely on `QObject` ownership for auto-disconnect.

Both solve the same problems — pick based on the rest of your stack.

<br>

#### Q3: "Should I use `weak_ptr` or a Subscription token?"

| Token | `weak_ptr` |
|-------|-----------|
| Subscriber doesn't have to be heap-allocated | Subscriber must be `shared_ptr`-managed |
| Explicit unsubscribe — caller controls lifetime | Implicit — observer just dies, subject prunes |
| Works for free functions and lambdas | Awkward for non-class callbacks |

Default to tokens. Add `weak_ptr` integration when observers are already `shared_ptr`-owned UI widgets/components.

<br>

#### Q4: "Is the Observer pattern an anti-pattern in concurrent code?"

It's not an anti-pattern, but it's full of footguns:
- Lock during emit → deadlock
- No lock → data race on listener list
- Async emit → ordering and lifetime get harder
- Callback runs on subject's thread → may block I/O loops

Solve all of these by **snapshotting under the lock and invoking unlocked**, and document which thread callbacks run on.

<br>

#### Q5: "How do I unit-test code that uses Observer?"

Inject the emitter as a dependency. In tests, subscribe a recording observer:

```cpp
std::vector<double> received;
auto id = stock.subscribe([&](double p) { received.push_back(p); });
stock.onTick(101.5);
stock.onTick(102.0);
assert(received == std::vector<double>{101.5, 102.0});
```

For async emit, add a "wait for N events" helper using `std::condition_variable`.

<br>

#### Q6: "What if an observer's callback throws?"

The behavior is up to the emitter. Two policies:
- **Stop on first throw** — propagate to the emit caller, abort delivery to remaining observers
- **Catch and continue** — log/swallow per callback, deliver to all

The "continue" policy is usually correct for pub/sub (one buggy subscriber shouldn't break others). Wrap each invocation:

```cpp
for (const auto& l : snapshot) {
    try { l.cb(data); }
    catch (const std::exception& e) {
        std::cerr << "observer " << l.id << " threw: " << e.what() << "\n";
    }
}
```

<br>

#### Q7: "How is Observer different from Mediator or Pub/Sub?"

| Pattern | Topology | Coupling |
|---------|----------|----------|
| **Observer** | One subject, many observers, **direct** coupling subject→observer-list | Tight |
| **Mediator** | Components talk through a central mediator that routes messages | Loose |
| **Pub/Sub** | Publishers and subscribers fully decoupled via a broker (often network) | Looser |

Our `EventEmitter` (named events, no direct subject) sits between Observer and Pub/Sub — it's an in-process bus.

<br>

#### Q8: "Can I cancel emit halfway through?"

Add a return type to the callback (`bool` for "continue?", or a `void(Event&)` where the event has `event.stopPropagation()`):

```cpp
struct Event { bool stopped = false; void stop() { stopped = true; } };
using Callback = std::function<void(Event&, const std::any&)>;

void emit(const std::string& event, const std::any& data) {
    Event e;
    for (const auto& l : snapshot) {
        l.cb(e, data);
        if (e.stopped) break;
    }
}
```

DOM events use exactly this pattern.

<br><br>

---

## Reflection Questions

1. What problems does Observer solve that direct method calls don't?
2. Why is subscription **lifetime** the hardest part of implementing Observer?
3. Why must the listener list be snapshotted before invoking callbacks during emit?
4. When would you choose `weak_ptr`-based observers over RAII subscription tokens?
5. Why is holding a lock during callback invocation dangerous?
6. How does the modern `std::function`-based variant differ from the GoF interface-based variant?

---

## Interview Questions

1. "Implement an `EventEmitter` with `on`, `emit`, `off`. Make it safe against unsubscribe-during-emit."
2. "What happens if an observer is destroyed before unsubscribing? How do you protect against it?"
3. "How would you make your emitter thread-safe? Where does the mutex go, and what's released before invoking callbacks?"
4. "Walk through the difference between Observer, Mediator, and Pub/Sub."
5. "How would you handle exceptions thrown by an observer's callback?"
6. "What's the difference between an interface-based observer and a `std::function`-based one? Trade-offs?"
7. "Implement `once(event, cb)` — fires at most once, then auto-unsubscribes."
8. "Design a typed event system (e.g., `on<UserLogin>(...)`) with no `std::any`."
9. "What ordering guarantees does your emit provide for a single event? Across events?"
10. "How would you implement async emission via a thread pool? What new failure modes appear?"

---

**Next**: Day 19 — Strategy & Command →
