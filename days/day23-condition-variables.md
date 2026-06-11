# Day 23: Condition Variables

[← Back to Study Plan](../lld-study-plan.md) | [← Day 22](day22-threads-mutexes.md)

> **Time**: ~1.5-2 hours
> **Goal**: A mutex protects shared data, but it can't *wait* for the data to reach a desired state without burning the CPU. A **condition variable** lets a thread sleep until another thread signals "something changed — go check." Learn the canonical mutex + condition-variable + predicate pattern, why you must *always* re-check the predicate (spurious and stolen wakeups), the difference between `notify_one` and `notify_all`, and the lost-wakeup bug that destroys signaling code. Build a **Bounded Blocking Queue** that blocks producers on full and consumers on empty — the backbone of every producer-consumer system and thread pool.

---
---

# PART 1: WHY CONDITION VARIABLES EXIST

---
---

<br>

<h2 style="color: #2980B9;">📘 23.1 The Problem: Waiting for State</h2>

A consumer wants the next item from a shared queue. With only a mutex, its options are bad:

```cpp
// BUSY-WAIT (spin) — burns a whole CPU core doing nothing
while (true) {
    std::lock_guard lk(m);
    if (!queue.empty()) { item = queue.front(); queue.pop(); break; }
    // else loop again immediately, re-locking, re-checking, frying the CPU
}
```

Spinning re-locks the mutex millions of times per second to check a condition that may not change for seconds. It wastes energy, starves other threads of the lock, and scales terribly.

The naive fix — `sleep` between checks — trades CPU waste for *latency*: you either sleep too long (slow to react) or too short (still wasteful). There is no good sleep duration.

What you actually want is: **"put this thread to sleep until someone tells me the state might have changed, then wake up and check."** That is exactly what a condition variable provides.

```
Producer                         Consumer
--------                         --------
                                 lock, queue empty → cv.wait() → SLEEP (lock released)
lock, push(item)
cv.notify_one()  ───────────────► wakes, re-acquires lock, re-checks: not empty → take item
unlock
```

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 23.2 The Three Ingredients</h2>

A condition variable never works alone. The pattern always involves **three** cooperating pieces:

1. **A mutex** — protects the shared state.
2. **The shared state itself** — the thing you're waiting *on* (e.g., the queue, a `bool ready`).
3. **A `std::condition_variable`** — the waiting/signaling channel.

```cpp
std::mutex m;                     // (1) protects ...
std::queue<int> q;                // (2) ... this shared state
std::condition_variable cv;       // (3) wait/notify channel
```

The condition variable does **not** store the condition. It is just a wait queue of sleeping threads. *You* own the actual condition (the predicate), and *you* are responsible for checking it. The CV's only jobs are "atomically release the lock and sleep" and "wake sleepers when signalled."

<br><br>

---
---

# PART 2: THE WAIT/NOTIFY PROTOCOL

---
---

<br>

<h2 style="color: #2980B9;">📘 23.3 wait() — Atomic Release-and-Sleep</h2>

`cv.wait(lock)` does three things, and the **atomicity** of the first two is the whole point:

1. **Atomically** unlock the mutex *and* put the thread to sleep on the CV.
2. (Sleeping... another thread can now take the lock, change state, and notify.)
3. On wakeup, **re-acquire** the lock before `wait` returns.

```cpp
std::unique_lock<std::mutex> lk(m);   // MUST be unique_lock, not lock_guard
cv.wait(lk);                          // releases m + sleeps; on wake, re-locks m
// here, m is held again
```

It must take a `std::unique_lock` (not `lock_guard`) because `wait` needs to unlock and relock — operations `lock_guard` doesn't support.

Why does the unlock-and-sleep need to be atomic? If `wait` unlocked first and *then* slept as two separate steps, a notifier could squeeze in between them, fire its `notify`, and the sleeper would miss it and sleep forever. The atomic combination closes that window — a notify that happens after the lock is released is guaranteed to be seen.

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 23.4 Spurious & Stolen Wakeups → Always Loop</h2>

A waiting thread can wake up even when *nobody notified it*. This is a **spurious wakeup**, and it is allowed by the standard (it stems from how OS futexes and signal handling work). Separately, a **stolen wakeup** happens when you were notified but another thread grabbed the resource before you re-acquired the lock — so by the time you wake, the condition is false again.

The defense is identical for both: **never assume the condition is true just because `wait` returned. Re-check it in a loop.**

```cpp
// CORRECT — re-check predicate after every wake
std::unique_lock<std::mutex> lk(m);
while (q.empty()) {        // loop, not if!
    cv.wait(lk);
}
int item = q.front(); q.pop();
```

```cpp
// WRONG — assumes wakeup means condition is true
std::unique_lock<std::mutex> lk(m);
if (q.empty()) {           // BUG: spurious/stolen wakeup → q still empty → crash on front()
    cv.wait(lk);
}
int item = q.front(); q.pop();
```

<br>

#### The predicate overload hides the loop for you

The two-argument form bakes the `while` loop in. **Always prefer it** — it is impossible to forget the loop:

```cpp
std::unique_lock<std::mutex> lk(m);
cv.wait(lk, [&]{ return !q.empty(); });   // equivalent to: while(!pred) wait(lk);
int item = q.front(); q.pop();
```

Read it as: "wait until the predicate is true." On every wakeup (spurious or real), it re-evaluates the lambda while holding the lock; only when the predicate returns `true` does `wait` return.

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 23.5 notify_one vs notify_all</h2>

| | `notify_one()` | `notify_all()` |
|---|---|---|
| Wakes | one waiting thread (unspecified which) | every waiting thread |
| Use when | any single waiter can handle the event (one item produced → one consumer) | the change may satisfy many waiters, or different waiters wait on different predicates |
| Cost | cheap | a thundering herd — all wake, all contend for the lock, all but the satisfied ones go back to sleep |

**Rule of thumb:** use `notify_one` when exactly one waiter should proceed (e.g., you pushed one item). Use `notify_all` when the state change might let *several* waiters proceed, or when waiters are waiting on *different* conditions that share one CV (e.g., "shutting down — everyone wake up and exit").

<br>

#### When notify_one is a subtle bug

If multiple threads wait on the same CV but with **different predicates**, `notify_one` can wake the "wrong" thread — one whose predicate is still false. It goes back to sleep, and the thread that *could* have proceeded never gets woken. Symptom: a hang under load. Either use `notify_all`, or use separate condition variables per predicate (as the bounded queue does — one CV for "not full," one for "not empty").

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 23.6 The Lost Wakeup Bug</h2>

A **lost wakeup** is the deadliest CV bug: a notify fires while no one is waiting (or in the gap before they start waiting), so it is silently dropped. The notification is *not* saved — a CV has no memory. If the predicate was already changed but the waiter checks *after* the notify, all is well (it sees the predicate is true and doesn't wait). The danger is notifying *without* holding the lock and *without* having set the state first:

```cpp
// BUG: notify can be lost
// Producer:
q.push(item);          // (state changed, but NOT under the lock!)
cv.notify_one();       // if consumer is between checking empty() and wait(), it misses this
```

```
Consumer thread                     Producer thread
---------------                     ---------------
lock
check empty() → true
                                    push(item)        (no lock held — race on q anyway!)
                                    notify_one()      ← fires into the void
cv.wait(lk)   ← sleeps forever, the notify already happened
```

<br>

#### The two rules that prevent lost wakeups

1. **Mutate the shared state while holding the mutex** (so it can't change in the gap between the waiter's check and its `wait`).
2. The waiter uses the **predicate loop** so that if the state was already satisfied before it called `wait`, it never sleeps at all.

```cpp
// CORRECT
{
    std::lock_guard lk(m);
    q.push(item);          // state change UNDER the lock
}
cv.notify_one();           // safe: waiter either already saw the item, or is genuinely asleep and will be woken
```

Holding the lock while changing state means a waiting consumer is *either* still before its predicate check (it will see the item and not wait) *or* genuinely asleep inside `wait` (the notify will reach it). There is no third gap.

> **Notify inside or outside the lock?** Both are correct *if* you changed the state under the lock. Notifying *outside* the lock can be marginally faster (the woken thread doesn't immediately block on a lock you still hold). Notifying *inside* is simpler to reason about. The non-negotiable rule is the *state change* under the lock, not the notify.

<br><br>

---
---

# PART 3: PRODUCER-CONSUMER & BACKPRESSURE

---
---

<br>

<h2 style="color: #2980B9;">📘 23.7 Why Bound the Queue?</h2>

An *unbounded* queue lets producers run arbitrarily far ahead of consumers. If producers are faster, the queue grows without limit until you run out of memory. A **bounded** queue applies **backpressure**: when the queue is full, producers *block* until a consumer makes room. This naturally throttles producers to the rate consumers can sustain.

```
Producers (fast)          Bounded queue (cap = 4)        Consumers (slow)
─────────────────►        ┌──┬──┬──┬──┐                  ─────────────────►
push() BLOCKS when full → │  │  │  │  │ ← pop() BLOCKS when empty
                          └──┴──┴──┴──┘
                          full  → producer waits on cv_notFull
                          empty → consumer waits on cv_notEmpty
```

This is the foundation of thread pools (task queue), pipelines (stage-to-stage buffers), and logging systems (log-line buffer). Backpressure is what keeps a system from melting down under overload.

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 23.8 Two Condition Variables, Two Predicates</h2>

Producers and consumers wait on *different* conditions:

- Producers wait for **not full** (`size < capacity`).
- Consumers wait for **not empty** (`size > 0`).

Using **two separate CVs** (one per predicate) lets `push` wake exactly a waiting consumer and `pop` wake exactly a waiting producer, with `notify_one` — no thundering herd, no waking-the-wrong-kind-of-waiter. A `push` makes the queue non-empty, so it notifies `cv_notEmpty`; a `pop` makes the queue non-full, so it notifies `cv_notFull`.

```
push():  wait on cv_notFull (until not full) → enqueue → notify cv_notEmpty
pop() :  wait on cv_notEmpty (until not empty) → dequeue → notify cv_notFull
```

You *could* use one CV with `notify_all`, but two CVs + `notify_one` is cleaner and avoids needless wakeups.

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 23.9 Graceful Shutdown</h2>

A blocking queue must support shutdown, or consumers blocked in `pop()` will sleep forever when production stops. The pattern: a `closed_` flag, set under the lock, plus `notify_all` to wake *everyone* so they can observe the flag and exit.

```cpp
void close() {
    { std::lock_guard lk(m_); closed_ = true; }
    cv_notEmpty_.notify_all();   // wake all consumers so they can drain/exit
    cv_notFull_.notify_all();    // wake all producers so they can bail out
}
```

Consumers' predicate becomes "not empty **or** closed," and after waking they distinguish: if there's an item, take it; if empty and closed, return "no more items." This lets the queue **drain** remaining items before reporting end-of-stream.

<br><br>

---
---

# PART 4: BUILD EXERCISE

---
---

<br>

<h2 style="color: #2980B9;">📘 23.10 Exercise: Bounded Blocking Queue</h2>

Build a `BoundedQueue<T>` that blocks producers on full and consumers on empty, supports graceful close-and-drain, and run it under a producer-consumer stress test verifying that every produced item is consumed exactly once.

<br>

#### Skeleton (header)

```cpp
// bounded_queue.h
#pragma once
#include <mutex>
#include <condition_variable>
#include <queue>
#include <optional>
#include <cstddef>

template <typename T>
class BoundedQueue {
    mutable std::mutex m_;
    std::condition_variable cv_notFull_;
    std::condition_variable cv_notEmpty_;
    std::queue<T> q_;
    std::size_t capacity_;
    bool closed_ = false;

public:
    explicit BoundedQueue(std::size_t capacity) : capacity_(capacity) {}

    // Blocks while full. Returns false if the queue was closed (item not pushed).
    bool push(T item) {
        std::unique_lock<std::mutex> lk(m_);
        cv_notFull_.wait(lk, [&]{ return q_.size() < capacity_ || closed_; });
        if (closed_) return false;
        q_.push(std::move(item));
        lk.unlock();                 // unlock before notify (optional optimization)
        cv_notEmpty_.notify_one();   // a consumer can now proceed
        return true;
    }

    // Blocks while empty. Returns nullopt only when the queue is closed AND drained.
    std::optional<T> pop() {
        std::unique_lock<std::mutex> lk(m_);
        cv_notEmpty_.wait(lk, [&]{ return !q_.empty() || closed_; });
        if (q_.empty()) {            // implies closed_ — nothing left to drain
            return std::nullopt;
        }
        T item = std::move(q_.front());
        q_.pop();
        lk.unlock();
        cv_notFull_.notify_one();    // a producer can now proceed
        return item;
    }

    // Non-blocking variants (useful for try-style callers).
    bool tryPush(T item) {
        std::lock_guard<std::mutex> lk(m_);
        if (closed_ || q_.size() >= capacity_) return false;
        q_.push(std::move(item));
        cv_notEmpty_.notify_one();
        return true;
    }

    std::optional<T> tryPop() {
        std::lock_guard<std::mutex> lk(m_);
        if (q_.empty()) return std::nullopt;
        T item = std::move(q_.front());
        q_.pop();
        cv_notFull_.notify_one();
        return item;
    }

    // Wake everyone; no more pushes accepted, but consumers can still drain.
    void close() {
        { std::lock_guard<std::mutex> lk(m_); closed_ = true; }
        cv_notEmpty_.notify_all();
        cv_notFull_.notify_all();
    }

    std::size_t size() const {
        std::lock_guard<std::mutex> lk(m_);
        return q_.size();
    }
};
```

<br>

#### Test driver

```cpp
// main.cpp
#include "bounded_queue.h"
#include <thread>
#include <vector>
#include <atomic>
#include <iostream>
#include <cassert>
#include <numeric>

int main() {
    std::cout << "=== Bounded blocking queue: producer-consumer ===\n";

    constexpr int kProducers = 4;
    constexpr int kConsumers = 4;
    constexpr int kItemsPerProducer = 50000;
    constexpr int kTotal = kProducers * kItemsPerProducer;
    constexpr std::size_t kCapacity = 16;   // small cap → producers WILL block on full

    BoundedQueue<int> queue(kCapacity);

    std::atomic<long long> producedSum{0};
    std::atomic<long long> consumedSum{0};
    std::atomic<int> consumedCount{0};

    std::vector<std::thread> producers, consumers;

    for (int p = 0; p < kProducers; ++p) {
        producers.emplace_back([&, p]{
            long long localSum = 0;
            for (int i = 0; i < kItemsPerProducer; ++i) {
                int value = p * kItemsPerProducer + i;   // unique values
                bool ok = queue.push(value);
                assert(ok);
                localSum += value;
            }
            producedSum += localSum;
        });
    }

    for (int c = 0; c < kConsumers; ++c) {
        consumers.emplace_back([&]{
            long long localSum = 0;
            int localCount = 0;
            while (auto item = queue.pop()) {   // nullopt → closed & drained → stop
                localSum += *item;
                ++localCount;
            }
            consumedSum += localSum;
            consumedCount += localCount;
        });
    }

    for (auto& t : producers) t.join();   // wait until all items produced
    queue.close();                        // now signal consumers to drain & stop
    for (auto& t : consumers) t.join();

    long long expectedSum = static_cast<long long>(kTotal - 1) * kTotal / 2;

    std::cout << "  produced count = " << kTotal
              << ", consumed count = " << consumedCount.load() << "\n";
    std::cout << "  produced sum   = " << producedSum.load()
              << ", consumed sum   = " << consumedSum.load()
              << " (expected " << expectedSum << ")\n";

    assert(consumedCount.load() == kTotal);          // every item consumed exactly once
    assert(producedSum.load() == expectedSum);
    assert(consumedSum.load() == expectedSum);       // no item lost, none double-counted
    assert(queue.size() == 0);

    std::cout << "All assertions passed.\n";
    return 0;
}
```

<br>

#### Compile and run

```bash
g++ -std=c++17 -Wall -Wextra -Wpedantic -O2 -pthread \
    -o day23 main.cpp && ./day23
```

#### Under ThreadSanitizer

```bash
g++ -std=c++17 -Wall -Wextra -Wpedantic -g -fsanitize=thread -pthread \
    -o day23_tsan main.cpp && ./day23_tsan
```

TSan should report nothing. The sum-and-count checks are a strong correctness oracle: if any item were lost, duplicated, or torn, the sums would not match.

<br>

#### Expected output

```
=== Bounded blocking queue: producer-consumer ===
  produced count = 200000, consumed count = 200000
  produced sum   = 19999900000, consumed sum   = 19999900000 (expected 19999900000)
All assertions passed.
```

<br>

#### Bonus Challenges

1. **Prove backpressure works.** Add a counter that records the max observed `q_.size()`. Assert it never exceeds `capacity_`. Then make producers much faster than consumers (add a `sleep` in the consumer) and confirm producers spend time blocked in `push` rather than the queue growing unbounded.

2. **Break it on purpose.** Replace the predicate-loop `wait(lk, pred)` with a bare `if + wait`. Run the stress test under heavy load; observe occasional crashes or lost items from spurious/stolen wakeups. Explain each failure.

3. **Timed wait.** Add `std::optional<T> popFor(std::chrono::milliseconds timeout)` using `cv.wait_for`. Return `nullopt` on timeout. Be careful: distinguish "timed out" from "closed and empty."

4. **One CV instead of two.** Re-implement with a single `condition_variable` and `notify_all`. Measure the extra wakeups (thundering herd) under contention versus the two-CV version. Explain the cost.

5. **Lost-wakeup demonstration.** Write a deliberately broken `push` that changes state *without* holding the lock, then notifies. Construct a timing (with strategic sleeps) where a consumer misses the notification and hangs. Then fix it and explain precisely which rule you restored.

6. **Multi-item ops.** Add `pushAll(std::vector<T>)` and a `popUpTo(n)` that drains up to *n* items in one lock acquisition. Discuss the latency-vs-throughput trade-off of batching, and which CV(s) to notify (`notify_one` vs `notify_all`) after a batch push.

<br><br>

---
---

# PART 5: DEEP DIVE — COMMON QUESTIONS

---
---

<br>

<h2 style="color: #2980B9;">📘 23.11 Q&A</h2>

<br>

#### Q1: "Why must I re-check the predicate in a loop instead of an `if`?"

Two reasons. **Spurious wakeups**: the OS may wake a waiter with no notification at all — the standard explicitly permits this. **Stolen wakeups**: you were notified, but another thread grabbed the resource before you re-acquired the lock, so the condition is false again by the time you wake. In both cases an `if` would proceed on a false condition (e.g., `pop` an empty queue → crash). The loop re-validates the condition after every wake. The `wait(lock, predicate)` overload bakes this loop in — always use it.

<br>

#### Q2: "Why does `wait` need a `unique_lock` and not a `lock_guard`?"

`wait` must *unlock* the mutex while sleeping and *relock* it on wakeup. `lock_guard` offers no unlock/relock operations; `unique_lock` does. The CV API only accepts `unique_lock` for exactly this reason.

<br>

#### Q3: "What is a lost wakeup and how do I prevent it?"

A lost wakeup is a `notify` that fires when no thread is waiting (or in the tiny gap before a thread starts waiting), so it's silently dropped — a CV has no memory of past notifications. Prevent it with two rules: (1) always change the shared state *while holding the mutex*, and (2) always wait with the predicate loop. Together these guarantee a waiter either sees the already-satisfied condition (and never sleeps) or is genuinely asleep when the notify arrives.

<br>

#### Q4: "notify_one or notify_all?"

`notify_one` when exactly one waiter should proceed (you produced one item → one consumer can take it). `notify_all` when the change could let multiple waiters proceed, when waiters wait on different predicates sharing one CV, or on shutdown ("everyone wake and exit"). `notify_all` is correct more often but can cause a thundering herd; `notify_one` is cheaper but can hang if it wakes a waiter whose predicate is still false.

<br>

#### Q5: "Should I notify with the lock held or released?"

Either is correct *provided you changed the state under the lock*. Notifying after releasing the lock can be slightly faster — the woken thread won't immediately block trying to acquire a lock you still hold. Notifying while holding it is simpler to reason about. The load-bearing rule is the state mutation under the lock; the notify position is an optimization detail.

<br>

#### Q6: "Why is busy-waiting (spinning) bad? Aren't spinlocks fast?"

A spinlock is fine when the wait is *guaranteed to be extremely short* (a few instructions) and you have spare cores — the wakeup latency beats putting a thread to sleep. But waiting on an arbitrary application condition (an item arriving, a file finishing) can take milliseconds or seconds; spinning then burns a full core doing nothing, starves other threads of the CPU, and wastes power. Condition variables let the OS deschedule the waiter entirely until there's real work.

<br>

#### Q7: "How do I cleanly shut down threads blocked in `pop()`?"

Add a `closed_` flag set under the lock, make the wait predicate "not empty **or** closed," and call `notify_all` on close so every blocked waiter wakes to observe the flag. Distinguish drain-then-stop (consume remaining items, then return end-of-stream when empty and closed) from immediate abort. Never just `detach` and hope — that races against process teardown.

<br>

#### Q8: "Can a notify wake the 'wrong' thread?"

Yes — if multiple threads wait on the *same* CV with *different* predicates, `notify_one` may wake one whose predicate is still false; it re-checks, finds nothing, and goes back to sleep, while the thread that could proceed never gets woken. Symptom: a hang. Fixes: use `notify_all`, or give each distinct predicate its own condition variable (as the bounded queue does with `cv_notFull` and `cv_notEmpty`).

<br><br>

---

## Reflection Questions

1. Why can't a mutex alone efficiently wait for a condition? What does a condition variable add?
2. Name the three ingredients every condition-variable pattern uses. Which one stores the actual condition?
3. Explain spurious and stolen wakeups. Why does the predicate loop defend against both?
4. Describe the lost-wakeup bug and the two rules that prevent it.
5. When is `notify_one` a bug, and how do separate condition variables fix it?
6. How does a bounded queue create backpressure, and why does that matter under overload?

---

## Interview Questions

1. "Implement a thread-safe bounded blocking queue with `push` and `pop`."
2. "What is a spurious wakeup? Why must you wait with a loop or a predicate?"
3. "Why does `condition_variable::wait` require a `unique_lock`?"
4. "Explain the lost-wakeup problem and how to prevent it."
5. "When do you use `notify_one` vs `notify_all`? Give a case where `notify_one` causes a hang."
6. "Why must you modify the shared state while holding the mutex, even though `notify` can be called after unlocking?"
7. "Implement a producer-consumer system. How do you apply backpressure?"
8. "How do you gracefully shut down consumers blocked in `pop()`? How do you drain remaining items first?"
9. "Why is busy-waiting bad? When is a spinlock actually appropriate?"
10. "Design a queue where producers and consumers wait on different conditions. Why two condition variables?"

---

**Next**: Day 24 — Semaphore & Barrier →
