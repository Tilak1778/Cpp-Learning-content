# Day 33: Actor Model

[← Back to Study Plan](../lld-study-plan.md) | [← Day 32](day32-lock-free-queue.md)

> **Time**: ~1.5-2 hours
> **Goal**: After three days of shared-memory concurrency (locks, atomics, lock-free queues), the actor model is the *opposite* philosophy: **share nothing, communicate by messages**. Each actor owns a private state and a thread, processes one message at a time from a mailbox, and never touches another actor's memory. Learn message-passing concurrency, one-thread-per-actor, typed mailboxes, supervision (overview), and exactly how this trades the bugs of shared memory for a different set. Build a working `Actor` class: its own thread, a message queue, `send(msg)`, and an `onReceive(handler)`.

---
---

# PART 1: THE PHILOSOPHY — SHARE NOTHING

---
---

<br>

<h2 style="color: #2980B9;">📘 33.1 The Core Idea</h2>

Carl Hewitt's actor model (1973) defines a unit of computation — an **actor** — that, in response to a message, can do exactly three things:

1. **Send** messages to other actors (whose addresses it knows).
2. **Create** new actors.
3. **Change its own behaviour/state** for the *next* message it handles.

Crucially, an actor has **private state that no other actor can touch**. There is no shared memory between actors. The only way to influence an actor is to *send it a message*; the actor processes messages **one at a time**, sequentially. That single rule eliminates an entire universe of bugs.

```
   ┌────────────┐   send(msg)    ┌────────────┐
   │  Actor A   │ ─────────────► │  Actor B   │
   │ ┌────────┐ │                │ ┌────────┐ │
   │ │private │ │                │ │mailbox │ │  ← messages queue here
   │ │ state  │ │                │ ├────────┤ │
   │ └────────┘ │ ◄───────────── │ │private │ │
   └────────────┘   send(reply)  │ │ state  │ │  ← only B touches B's state
                                  │ └────────┘ │
                                  └────────────┘
   No shared memory. No locks. The mailbox serializes access to state.
```

<br>

#### Why this kills concurrency bugs

Because only one thread (the actor's own) ever touches an actor's state, and it processes one message at a time, **there are no data races inside an actor — no locks needed, ever.** You traded "lock everything that's shared" for "share nothing." The mailbox *is* the synchronization: it serializes all access to the actor's state into a single sequential stream.

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 33.2 Actor Model vs Shared Memory</h2>

| | Shared memory (locks/atomics) | Actor model (messages) |
|---|------------------------------|------------------------|
| **State sharing** | Threads read/write shared data | Each actor owns private state; no sharing |
| **Synchronization** | Locks, atomics, memory ordering | The mailbox; one message at a time |
| **Primary bug class** | Data races, deadlocks, lost wakeups | Lost/duplicated messages, mailbox overflow, deadlock-by-message-cycle |
| **Communication cost** | Cheap (pointer to shared data) | Copy/move the message into the mailbox |
| **Mental model** | "Protect the data" | "Where does the message go next?" |
| **Scaling across machines** | Hard (shared memory is local) | Natural (a message can cross a network) |
| **Debugging** | Heisenbugs, timing-dependent | Inspect message logs; deterministic per-actor |

The actor model doesn't make concurrency *easy* — it makes it a **different problem**. You stop reasoning about who holds which lock and start reasoning about message flow, ordering, and what happens when a mailbox fills up or an actor dies. Erlang/OTP, Akka (Scala/Java), Microsoft Orleans, and C++'s CAF (C++ Actor Framework) are production actor systems.

<br>

#### "Don't communicate by sharing memory; share memory by communicating."

This Go proverb (Go uses channels + goroutines, a close cousin) captures the inversion. Instead of putting data in a shared place and guarding it, you put data *in a message* and hand ownership to whoever should act on it next. Ownership moves with the message — exactly the move-semantics discipline from Day 1, now applied across threads.

<br><br>

---
---

# PART 2: ANATOMY OF AN ACTOR

---
---

<br>

<h2 style="color: #2980B9;">📘 33.3 The Pieces</h2>

A minimal actor needs four things:

1. **A mailbox** — a thread-safe queue of messages (a `std::queue` + mutex + condition_variable, exactly the pattern from the Day 29 thread pool, *or* an SPSC ring from Day 32 if there's a single sender).
2. **A dedicated thread** — runs the actor's event loop: block until a message arrives, dequeue it, process it, repeat.
3. **A message handler** — the `onReceive` callback (or a `switch`/visitor over message types) that defines behaviour.
4. **Private state** — captured by the handler; touched *only* by the actor's own thread, so no synchronization is needed.

```
Actor's event loop (runs on the actor's own thread):
    for (;;) {
        Message m = mailbox.take();   // blocks if empty
        if (m is Stop) break;         // poison-pill shutdown
        handler(m);                   // mutate private state freely — no lock!
    }
```

The event loop is the entire concurrency model. Because `handler(m)` runs only on this one thread, and messages are taken one at a time, the handler can mutate state with zero locking. All the synchronization is concentrated in `mailbox.take()`.

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 33.4 Typed Mailboxes</h2>

A mailbox can be **untyped** (holds `std::any` or a base-class pointer) or **typed**. Untyped is flexible but pushes type-checking to runtime. Two common typed approaches in C++:

#### Approach A: `std::variant` of known message types

```cpp
struct Deposit  { double amount; };
struct Withdraw { double amount; };
struct GetBalance { std::promise<double> reply; };

using Message = std::variant<Deposit, Withdraw, GetBalance>;

// Handler dispatches with std::visit + overload set:
void handle(Message& msg, double& balance) {
    std::visit([&](auto&& m) {
        using M = std::decay_t<decltype(m)>;
        if constexpr (std::is_same_v<M, Deposit>)      balance += m.amount;
        else if constexpr (std::is_same_v<M, Withdraw>) balance -= m.amount;
        else if constexpr (std::is_same_v<M, GetBalance>) m.reply.set_value(balance);
    }, msg);
}
```

This is type-safe, allocation-free for the message, and the compiler checks you handle every variant case (with an `overloaded` visitor). The downside: every actor must know all its message types up front.

#### Approach B: type-erased `std::function<void()>` ("active object")

Each "message" is a closure that captures its arguments and operates on the actor's state. This is the **Active Object** pattern — the actor exposes methods that *enqueue work* instead of doing it inline:

```cpp
actor.send([&state]{ state.deposit(100); });   // the message IS the work
```

Flexible (any operation), but loses the explicit message taxonomy and the compile-time exhaustiveness check. We use this style in the build because it's the most general and mirrors the thread-pool task queue you already built.

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 33.5 Request/Reply with Futures</h2>

`send()` is fire-and-forget (asynchronous). But you often need a *reply* — "what's the balance?" The idiom: include a `std::promise` in the message; the actor fulfils it; the caller waits on the matching `std::future`.

```
caller                          actor thread
──────                          ────────────
promise<double> p
future<double> f = p.get_future()
send(GetBalance{ move(p) }) ──► mailbox ──► handler: m.reply.set_value(balance)
f.get()  ◄──────────────────────────────────  (future becomes ready)
```

This bridges the async actor world back to synchronous calling code. It's the same `packaged_task`/promise mechanism from Day 27 and the thread pool — the actor just happens to be the thing that fulfils the promise. Beware: blocking on `f.get()` inside *another* actor's handler can deadlock if that actor needs to process a message to produce the reply (a message cycle).

<br><br>

---
---

# PART 3: SUPERVISION & FAILURE (OVERVIEW)

---
---

<br>

<h2 style="color: #2980B9;">📘 33.6 "Let It Crash"</h2>

Erlang's famous philosophy: don't write defensive code for every possible error inside an actor — let the actor **crash**, and have a **supervisor** actor decide what to do (restart it, restart it with fresh state, escalate, or stop). Errors are handled *structurally* (in the supervision tree) rather than *inline* (try/catch everywhere).

```
        ┌─────────────┐
        │ Supervisor  │   strategy: one-for-one / one-for-all / rest-for-one
        └──────┬──────┘
        ┌──────┼──────┐
        ▼      ▼      ▼
   ┌───────┐┌───────┐┌───────┐
   │Worker1││Worker2││Worker3│   ← if Worker2 crashes, supervisor restarts it
   └───────┘└───────┘└───────┘     (its private state is gone — fresh start)
```

| Strategy | On a child's failure |
|----------|----------------------|
| **one-for-one** | Restart only the failed child |
| **one-for-all** | Restart *all* children (they're interdependent) |
| **rest-for-one** | Restart the failed child and those started after it |

Because actors share no state, a crashed actor can be discarded and replaced cleanly — there's no half-mutated shared structure to repair (the disaster of a thread crashing while holding a lock). This is why "let it crash" is *safe* in actor systems and *terrifying* in shared-memory ones. Supervision trees give you fault isolation and self-healing. (Full supervision is beyond today's build — we implement a clean shutdown and note where a supervisor would hook in.)

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 33.7 Backpressure and Mailbox Bounds</h2>

The mailbox is a queue, and queues can grow without bound if a producer outpaces the actor. An unbounded mailbox can exhaust memory; a bounded one must decide what to do when full:

| Policy | Behaviour | Use when |
|--------|-----------|----------|
| **Block sender** | `send()` waits for space | You want flow control / backpressure |
| **Drop newest** | Discard the incoming message | Telemetry; losing some is fine |
| **Drop oldest** | Evict the head to make room | "Latest value wins" (sensor readings) |
| **Fail / throw** | Reject the send | Caller must handle overload |

This mirrors the thread-pool backpressure discussion (Day 29 bonus). The actor model doesn't escape the fundamental queueing-theory truth: if arrival rate > service rate forever, *something* must give. Choosing the policy is a design decision, not an afterthought.

<br><br>

---
---

# PART 4: BUILD EXERCISE

---
---

<br>

<h2 style="color: #2980B9;">📘 33.8 Exercise: An Actor Class</h2>

Build an `Actor` with its own thread, a thread-safe mailbox, `send()`, and a settable `onReceive` handler. Use the type-erased (Active Object) style so the message is a closure operating on the actor's private state. Demonstrate that the handler never needs a lock, and add request/reply via futures.

<br>

#### Skeleton

```cpp
// actor.h
#pragma once
#include <condition_variable>
#include <functional>
#include <future>
#include <mutex>
#include <queue>
#include <thread>
#include <utility>

class Actor {
public:
    using Message = std::function<void()>;   // message = work to run on actor's state

    Actor() : m_thread([this] { run(); }) {}

    ~Actor() {
        stop();                              // graceful: drain then exit
        if (m_thread.joinable()) m_thread.join();
    }

    Actor(const Actor&)            = delete;
    Actor& operator=(const Actor&) = delete;

    // Asynchronous fire-and-forget send. Thread-safe; any thread may call.
    void send(Message msg) {
        {
            std::lock_guard<std::mutex> lk(m_mutex);
            if (m_stopped) return;           // ignore sends after stop
            m_mailbox.push(std::move(msg));
        }
        m_cv.notify_one();
    }

    // Request/reply: returns a future fulfilled by the actor thread.
    template <typename F>
    auto ask(F&& f) -> std::future<std::invoke_result_t<F>> {
        using R = std::invoke_result_t<F>;
        auto prom = std::make_shared<std::promise<R>>();
        auto fut  = prom->get_future();
        send([prom, f = std::forward<F>(f)]() mutable {
            try {
                if constexpr (std::is_void_v<R>) { f(); prom->set_value(); }
                else                              { prom->set_value(f()); }
            } catch (...) {
                prom->set_exception(std::current_exception());
            }
        });
        return fut;
    }

    void stop() {
        {
            std::lock_guard<std::mutex> lk(m_mutex);
            if (m_stopped) return;
            m_stopped = true;
        }
        m_cv.notify_one();
    }

private:
    void run() {
        for (;;) {
            Message msg;
            {
                std::unique_lock<std::mutex> lk(m_mutex);
                m_cv.wait(lk, [this] { return m_stopped || !m_mailbox.empty(); });
                if (m_stopped && m_mailbox.empty()) return;   // drain, then exit
                msg = std::move(m_mailbox.front());
                m_mailbox.pop();
            }
            msg();        // run on the actor's thread — touches private state, NO LOCK
        }
    }

    std::queue<Message>     m_mailbox;
    std::mutex              m_mutex;
    std::condition_variable m_cv;
    bool                    m_stopped = false;
    std::thread             m_thread;     // declared last: started after everything is built
};
```

> **Member-order gotcha**: `m_thread` is declared **last** so it's constructed *after* the mutex/cv/mailbox. The worker `run()` touches those, so they must exist before the thread starts. (Members are initialized in declaration order, regardless of the initializer-list order.)

<br>

#### A bank-account actor + test driver

```cpp
// main.cpp
#include "actor.h"
#include <atomic>
#include <cassert>
#include <iostream>
#include <vector>

int main() {
    std::cout << "=== 1. No-lock state mutation via messages ===\n";
    {
        Actor account;
        // Private state owned by the actor; only the actor thread touches it.
        // We keep it here and only mutate it from inside messages.
        double balance = 0.0;

        // 1000 concurrent deposits from many threads, but each runs serially
        // inside the actor — so 'balance' needs NO lock.
        constexpr int kThreads = 8, kPerThread = 1000;
        std::vector<std::thread> senders;
        for (int t = 0; t < kThreads; ++t)
            senders.emplace_back([&account, &balance] {
                for (int i = 0; i < kPerThread; ++i)
                    account.send([&balance] { balance += 1.0; });  // no lock!
            });
        for (auto& s : senders) s.join();

        // Ask for the final balance (synchronizes with all prior sends from
        // this thread? No — sends from OTHER threads must have completed; the
        // joins above guarantee all sends are enqueued. ask() runs after them.)
        double final = account.ask([&balance] { return balance; }).get();
        std::cout << "  final balance = " << final << "\n";
        assert(final == static_cast<double>(kThreads * kPerThread));
    }

    std::cout << "=== 2. Request/reply with futures ===\n";
    {
        Actor counter;
        int n = 0;
        counter.send([&n] { n += 10; });
        counter.send([&n] { n *= 2; });
        int result = counter.ask([&n] { return n; }).get();   // (0+10)*2 = 20
        std::cout << "  counter result = " << result << "\n";
        assert(result == 20);
    }

    std::cout << "=== 3. Messages are processed in send order (FIFO) ===\n";
    {
        Actor a;
        std::vector<int> log;
        for (int i = 0; i < 100; ++i)
            a.send([&log, i] { log.push_back(i); });
        a.ask([]{ return 0; }).get();        // barrier: waits for all prior msgs
        bool ordered = true;
        for (int i = 0; i < 100; ++i)
            if (log[i] != i) ordered = false;
        std::cout << "  in_order = " << (ordered ? "yes" : "NO") << "\n";
        assert(ordered);
    }

    std::cout << "=== 4. Exception in a message surfaces via ask().get() ===\n";
    {
        Actor a;
        auto fut = a.ask([]() -> int { throw std::runtime_error("oops"); });
        bool caught = false;
        try { fut.get(); }
        catch (const std::exception& e) {
            caught = true;
            std::cout << "  caught: " << e.what() << "\n";
        }
        assert(caught);
        // Actor still alive afterward:
        assert(a.ask([]{ return 99; }).get() == 99);
    }

    std::cout << "All assertions passed.\n";
    return 0;
}
```

<br>

#### Compile and run

```bash
g++ -std=c++17 -Wall -Wextra -Wpedantic -fsanitize=thread -pthread \
    -o day33 main.cpp && ./day33
```

ThreadSanitizer must report **no** races on `balance`/`n`/`log` — even though they're plain (non-atomic) variables — *because* every access happens on the single actor thread, serialized by the mailbox. That's the whole lesson: the actor turns shared mutable state into single-threaded state.

<br>

#### Expected output

```
=== 1. No-lock state mutation via messages ===
  final balance = 8000
=== 2. Request/reply with futures ===
  counter result = 20
=== 3. Messages are processed in send order (FIFO) ===
  in_order = yes
=== 4. Exception in a message surfaces via ask().get() ===
  caught: oops
All assertions passed.
```

<br>

#### Bonus Challenges

1. **Typed mailbox** — replace `std::function<void()>` with a `std::variant<Deposit, Withdraw, GetBalance>` and dispatch with `std::visit` + an `overloaded` visitor. Compare type-safety and ergonomics with the closure style.

2. **Bounded mailbox + backpressure** — cap the mailbox; make `send()` block on a second condition variable when full. Then add a `try_send()` that returns false instead of blocking.

3. **Actor addresses & a registry** — give each actor an ID; build a `system.spawn()` / `system.send(id, msg)` so actors reference each other by handle, not raw pointer (a step toward location transparency).

4. **Supervisor** — implement a supervisor actor that spawns N workers; if a worker's handler throws (catch it in `run()`), restart the worker with fresh state (one-for-one strategy). Log restarts.

5. **Ping-pong benchmark** — two actors bounce a message back and forth N times; measure messages/sec. Then compare to a mutex-protected shared counter doing N increments. Discuss where each wins.

6. **Avoid the ask-deadlock** — construct a scenario where actor A's handler calls `B.ask(...).get()` while B is busy and B's reply needs A. Show the deadlock, then fix it by making the reply a *message back to A* instead of a blocking `get()`.

<br><br>

---
---

# PART 5: DEEP DIVE — COMMON QUESTIONS

---
---

<br>

<h2 style="color: #2980B9;">📘 33.9 Q&A</h2>

<br>

#### Q1: "Why does an actor's state need no locks?"

Because exactly one thread — the actor's own — ever touches that state, and it processes one message at a time. There is no concurrent access, so there is no data race, so there is nothing for a lock to protect. The mailbox *is* the synchronization: it funnels all access into a single sequential stream on one thread. This is the central payoff of the model.

<br>

#### Q2: "How is the actor model different from a thread pool?"

A thread pool has *shared* worker threads pulling from a *shared* queue; tasks may run on any thread and typically operate on shared data (needing locks). An actor has *its own* thread and *private* state; messages to one actor are always processed by that actor's single thread, serially. Thread pool = "where does this task run?" (any free worker). Actor = "this entity owns its state and processes its mail one at a time." You can *implement* actors on top of a pool (one logical mailbox scheduled onto pool threads, ensuring no two of an actor's messages run concurrently), but the guarantee differs.

<br>

#### Q3: "Does the actor model eliminate deadlock?"

No — it changes its shape. There are no lock-ordering deadlocks (there are no locks), but you can deadlock via **message cycles**: A blocks on a reply from B, B blocks on a reply from A. Synchronous `ask().get()` between actors is the usual culprit. The fix is to stay asynchronous — reply with a *message* rather than blocking on a future. Mailbox overflow with blocking sends can also wedge a system.

<br>

#### Q4: "Why copy/move messages instead of sharing pointers?"

To preserve the share-nothing invariant. If two actors held a pointer to the same mutable object, you'd be back to shared-memory races and would need locks. Moving the data *into* the message transfers ownership to the recipient — only the recipient touches it afterward. (Immutable shared data is safe to share by pointer, since no one writes it; that's a common optimization.)

<br>

#### Q5: "What does 'let it crash' mean and why is it safe here but not with locks?"

It means: don't defensively handle every error inside an actor; let it fail and have a supervisor restart it. It's safe because a crashed actor shares no state — discarding and recreating it leaves no half-mutated structures behind. With locks, a thread that crashes (or throws) while holding a lock leaves shared data inconsistent and the lock possibly held forever — far harder to recover from cleanly.

<br>

#### Q6: "Why must `m_thread` be the last-declared member?"

Members are initialized in **declaration order**. The worker function `run()` immediately uses the mutex, condition variable, and mailbox, so those must be fully constructed *before* the thread starts. Declaring `m_thread` last guarantees it's constructed (and thus launched) after its dependencies. Declaring it first would start the thread before the mutex exists → undefined behaviour.

<br>

#### Q7: "How do I get a result back from an asynchronous actor?"

Embed a `std::promise` in the request and return its `std::future` to the caller (the `ask()` method does this). The actor fulfils the promise inside its handler; the caller blocks on `future.get()`. This bridges async message passing to synchronous code. Just don't `get()` from inside another actor's handler if it could create a message cycle (see Q3).

<br>

#### Q8: "When should I choose actors over shared memory?"

Actors shine when: state is naturally partitioned into independent entities (connections, sessions, game objects); you want fault isolation ("let it crash" + supervision); you may need to scale across machines (messages serialize over a network for free); or the team struggles with lock-based correctness. Shared memory wins when: communication is extremely hot and copying messages is too costly; the data is genuinely shared and fine-grained; or you need the absolute lowest latency (a lock-free structure beats a mailbox round-trip). Many systems mix both.

<br><br>

---

## Reflection Questions

1. State the three things an actor can do in response to a message. Why does "change its own behaviour" need no lock?
2. Explain precisely why an actor's private state is data-race-free without any synchronization on the state itself.
3. Contrast the primary bug classes of shared-memory vs actor concurrency.
4. How does request/reply work on top of fire-and-forget `send()`? Where can it deadlock?
5. Why is "let it crash" safe in an actor system but dangerous with locks?
6. What are the policy options when a bounded mailbox fills up, and what does each cost?

---

## Interview Questions

1. "Explain the actor model. What are the three fundamental actor operations?"
2. "Why does an actor need no locks to protect its state?"
3. "Compare the actor model with shared-memory concurrency: synchronization, bug classes, scaling."
4. "Implement a minimal `Actor` with its own thread, a mailbox, and `send()`."
5. "How do you get a reply from an asynchronous actor? Implement request/reply."
6. "Does the actor model eliminate deadlock? Justify your answer."
7. "What is 'let it crash' and a supervision tree? Why is it safe in actor systems?"
8. "What happens when a mailbox fills up, and how do you choose a backpressure policy?"
9. "Why must messages be copied/moved rather than passed by shared pointer?"
10. "When would you pick actors over a lock-free data structure, and vice versa?"

---

**Next**: Day 34 — Parallel Algorithms →
