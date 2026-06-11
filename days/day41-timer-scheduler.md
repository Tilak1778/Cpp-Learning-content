# Day 41: Design a Timer / Scheduler

[← Back to Study Plan](../lld-study-plan.md) | [← Day 40](day40-rate-limiter.md)

> **Time**: ~2-3 hours (weekend day)
> **Goal**: A timer/scheduler runs callbacks at or after specified times — `schedule(cb, delay)`, `scheduleRepeating(cb, interval)`, `cancel(id)`. It's the backbone of timeouts, retries, heartbeats, and cron-like jobs, and it's a rich LLD problem combining a **min-heap of deadlines**, a dedicated **timer thread** that sleeps with `condition_variable::wait_until` (and re-arms when a sooner task is added), correct **cancellation**, and the recurring-vs-one-shot distinction. You'll also survey **hashed timing wheels** — the O(1) structure used when there are huge numbers of timers. Then you build a working scheduler with all three operations.

---
---

# PART 1: THE PROBLEM

---
---

<br>

<h2 style="color: #2980B9;">📘 41.1 What a Scheduler Must Do</h2>

The API:

```cpp
using TimerId = std::uint64_t;
TimerId schedule(std::function<void()> cb, milliseconds delay);            // one-shot
TimerId scheduleRepeating(std::function<void()> cb, milliseconds interval); // recurring
bool    cancel(TimerId id);                                                 // stop a timer
```

Requirements that make it interesting:

| Requirement | Why it's hard |
|-------------|---------------|
| Fire at the **earliest** deadline first | Need an ordered structure keyed by deadline |
| **Add** a task that's sooner than the current sleep | The timer thread must wake early and re-arm |
| **Cancel** a pending task | Removing an arbitrary element from a heap is O(n) — need a trick |
| **Recurring** tasks | Must re-insert with the next deadline after firing |
| Don't busy-wait | Sleep precisely until the next deadline, wake on change |
| Run callbacks without blocking the timer thread | A slow callback shouldn't delay every other timer |

<br>

#### The shape of the solution

```
   Producers                  Scheduler core                 Timer thread
 ┌───────────┐  schedule()  ┌────────────────────┐  wait_until ┌──────────────┐
 │ any thread│ ───────────► │  min-heap by        │  (earliest  │ pop due tasks│
 │           │   cancel()   │  deadline           │   deadline) │ → run cbs    │
 └───────────┘ ───────────► │  + mutex + condvar  │ ◄────────── │ re-arm sleep │
                            └────────────────────┘   notify     └──────────────┘
```

A **min-heap** keyed by deadline gives O(log n) insert and O(1) peek-at-earliest. One **timer thread** sleeps until the top's deadline (or until a sooner task arrives and notifies it), pops everything due, and runs the callbacks.

<br><br>

---
---

# PART 2: THE MIN-HEAP OF DEADLINES

---
---

<br>

<h2 style="color: #2980B9;">📘 41.2 Why a Min-Heap</h2>

We always need the **soonest** deadline next. A binary min-heap (`std::priority_queue` with a `greater` comparator) gives:

- **`top()`** — the earliest deadline, O(1). This is what the timer thread sleeps until.
- **`push()`** — insert a new task, O(log n).
- **`pop()`** — remove the earliest after firing, O(log n).

```
   Min-heap ordered by deadline (earliest at the root):

                  [t=100ms]
                  /        \
            [t=250ms]    [t=180ms]
            /      \
       [t=400ms] [t=300ms]

   timer thread sleeps until 100ms, fires that task, pops it,
   then sleeps until the new top (180ms), and so on.
```

```cpp
#include <queue>
#include <vector>

struct Task {
    std::chrono::steady_clock::time_point deadline;
    TimerId                               id;
    std::function<void()>                 cb;
    std::chrono::milliseconds             interval{0};  // 0 = one-shot; >0 = recurring
};

// Min-heap: smallest deadline on top.
struct TaskCompare {
    bool operator()(const Task& a, const Task& b) const {
        return a.deadline > b.deadline;   // `greater` → min-heap
    }
};

std::priority_queue<Task, std::vector<Task>, TaskCompare> heap;
```

> **Use `steady_clock`, not `system_clock`.** `system_clock` can jump (NTP adjustments, the user changing the wall clock), which would make a scheduled delay fire early or late. `steady_clock` is monotonic — it only moves forward at a constant rate — which is exactly what "fire 5 seconds from now" needs.

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 41.3 The Cancellation Problem</h2>

A heap supports removing the *top* in O(log n), but **removing an arbitrary element** (the one a `cancel(id)` refers to) is O(n) — you'd have to find it first. There are two standard solutions:

**Approach A — Lazy cancellation (the common choice).** Keep a `set<TimerId>` of cancelled ids. Leave cancelled tasks in the heap; when one bubbles to the top and is about to fire, check the cancelled set — if present, discard it instead of running it.

```cpp
std::unordered_set<TimerId> cancelled;

bool cancel(TimerId id) {
    cancelled.insert(id);   // O(1); the task is dropped when it surfaces
    return true;
}
// at fire time:
if (cancelled.count(task.id)) { cancelled.erase(task.id); /* skip */ continue; }
```

Pros: O(1) cancel, simple. Cons: a cancelled task lingers in the heap until its deadline (wasted memory if you cancel far-future tasks en masse). Mitigate by occasionally rebuilding the heap if the cancelled set grows large.

**Approach B — Active removal with an index.** Maintain a `map<TimerId, handle>` into a heap structure that supports decrease-key/erase (a Fibonacci or indexed binary heap). O(log n) cancel, but far more complex code. Rarely worth it.

Most schedulers use **lazy cancellation**. We'll use it.

<br><br>

---
---

# PART 3: THE TIMER THREAD

---
---

<br>

<h2 style="color: #2980B9;">📘 41.4 `wait_until` and the Re-Arm Pattern</h2>

The timer thread's job: sleep until the next deadline, but wake **early** if (a) a sooner task is scheduled, or (b) we're shutting down. The primitive is `condition_variable::wait_until`.

```cpp
std::unique_lock<std::mutex> lk(m_mtx);
while (m_running) {
    if (m_heap.empty()) {
        m_cv.wait(lk);                          // nothing scheduled → sleep until notified
    } else {
        auto next = m_heap.top().deadline;
        // Sleep until `next`, but wake if notified (new sooner task / shutdown).
        if (m_cv.wait_until(lk, next) == std::cv_status::timeout) {
            // Deadline reached → fire everything that is now due.
            fireDueTasks(lk);
        }
        // If NOT timeout, we were notified → loop re-reads the (possibly new) top.
    }
}
```

The mechanism, step by step:

1. Peek the earliest deadline `next`.
2. `wait_until(lk, next)` releases the mutex and sleeps until *either* `next` arrives (`timeout`) *or* someone calls `notify` (a new task or shutdown).
3. On **timeout** → the soonest task is due → fire all due tasks, re-insert recurring ones, loop.
4. On **notify** → the heap changed; loop back and recompute the new earliest deadline (which may now be sooner). This is the **re-arm**: a `schedule()` with a nearer deadline calls `m_cv.notify_one()`, waking the timer thread so it doesn't oversleep.

```
   timer thread sleeping until t=500ms
            │
   schedule(cb, 100ms) on another thread  → pushes task, notify_one()
            ▼
   wait_until returns (notified, not timeout) → loop re-reads top → now t=100ms
            │
   wait_until(100ms) → timeout → fire
```

Without the re-arm notify, a task scheduled sooner than the current sleep would only fire after the *old* (later) deadline — late. The notify is essential.

<br>

#### Why `wait_until` and not `sleep_for`

`sleep_for(duration)` can't be interrupted — if you sleep 500 ms and a 10 ms task arrives, you'd miss its deadline by 490 ms. `wait_until` on a condition variable *can* be woken early by `notify`, which is exactly the interruptibility we need. Also, `wait_until` takes an absolute time point, so it isn't skewed by the time spent computing/locking before the sleep.

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 41.5 Firing, Recurring, and Not Blocking the Thread</h2>

When deadlines come due, pop all due tasks, then run their callbacks. Two subtleties:

**1. Don't run callbacks while holding the lock.** A slow or re-entrant callback (one that calls `schedule()` again) would block the timer thread and risk deadlock. Collect due callbacks, **release the lock**, then run them.

```cpp
void fireDueTasks(std::unique_lock<std::mutex>& lk) {
    auto now = std::chrono::steady_clock::now();
    std::vector<Task> due;
    while (!m_heap.empty() && m_heap.top().deadline <= now) {
        Task t = m_heap.top();
        m_heap.pop();
        if (m_cancelled.count(t.id)) { m_cancelled.erase(t.id); continue; }  // lazy cancel
        if (t.interval.count() > 0) {                 // recurring → re-insert
            t.deadline = now + t.interval;
            m_heap.push(t);                            // (push a COPY before moving cb out)
        }
        due.push_back(std::move(t));
    }
    lk.unlock();
    for (auto& t : due) t.cb();                        // run callbacks WITHOUT the lock
    lk.lock();
}
```

**2. Recurring re-insertion.** A recurring task is re-pushed with `deadline = now + interval` *before* its callback runs (so the schedule stays steady). Note the ordering: re-insert the recurring task into the heap (which needs a copy of `cb`), then move the task into the `due` list to run. If a recurring task is cancelled, the lazy-cancel check skips both running and re-inserting it.

> **Fixed-rate vs fixed-delay recurrence.** `now + interval` schedules relative to *when it fired* (fixed-delay — drifts if callbacks are slow). For *fixed-rate* (fire every interval regardless of callback duration), store the *original* scheduled time and add `interval` to *that*: `t.deadline += t.interval`. Pick deliberately — heartbeats usually want fixed-rate; retry backoffs want fixed-delay.

<br>

> **Long-running callbacks.** Even with the lock released, a callback that takes longer than the next interval will delay subsequent timers (one timer thread runs them serially). For heavy callbacks, hand them off to a **thread pool** (`scheduler enqueues onto a pool`) so the timer thread only *triggers* work and immediately returns to timing.

<br><br>

---
---

# PART 4: HASHED TIMING WHEELS (OVERVIEW)

---
---

<br>

<h2 style="color: #2980B9;">📘 41.6 When the Heap Isn't Enough</h2>

A min-heap is O(log n) per insert/remove. With **hundreds of thousands** of concurrent timers (think: a connection-timeout per socket in a server with 1M connections), that log factor and the per-task allocation add up. The classic answer is the **hashed timing wheel** (Varghese & Lauck), used in the Linux kernel, Netty, Kafka, and many network stacks.

```
   A timing wheel = a circular array of "buckets", each a list of timers.
   A "current slot" pointer advances one bucket per tick.

        slot:  0    1    2    3    4    5    6    7
              [ ]  [●]  [ ]  [●●] [ ]  [ ]  [●]  [ ]
                    ▲
              current tick → fire everything in this bucket, advance

   To schedule a timer with delay d ticks:
       bucket = (current + d) mod wheelSize     ← O(1) insert
   Each tick: fire & clear the current bucket   ← O(1) amortized per timer
```

- **Insert** = compute a bucket index and append to its list → **O(1)**.
- **Per-tick expiry** = process one bucket's list → **O(1)** amortized per timer.
- **Cancel** = remove from the timer's bucket list (O(1) with an intrusive list node).

The trade-off is **time resolution**: timers fire at tick granularity (e.g., every 10 ms), not to-the-microsecond. For timeouts and heartbeats that's fine.

<br>

#### Hierarchical timing wheels

A single wheel only covers `wheelSize × tickDuration` of range. For longer delays, use **hierarchical** wheels (like a clock's seconds/minutes/hours hands): a "seconds" wheel, a "minutes" wheel, etc. When the seconds wheel wraps, it "cascades" timers down from the minutes wheel. This gives O(1) operations across an enormous time range with bounded memory — which is why the Linux kernel uses it.

| | Min-heap | Hashed timing wheel |
|---|----------|---------------------|
| **Insert** | O(log n) | **O(1)** |
| **Delete-min / tick** | O(log n) | **O(1)** amortized |
| **Cancel** | O(1) lazy / O(log n) active | **O(1)** |
| **Precision** | Exact deadline | Tick granularity |
| **Best for** | Modest timer counts, exact times | Huge timer counts, timeout-style timers |

For an interview LLD scheduler, build the **heap** version (clearer, exact) and *mention* timing wheels as the scaling answer. We do exactly that.

<br><br>

---
---

# PART 5: BUILD EXERCISE

---
---

<br>

<h2 style="color: #2980B9;">📘 41.7 Exercise: A Min-Heap Scheduler with a Timer Thread</h2>

Build a `Scheduler` with `schedule`, `scheduleRepeating`, `cancel`, and graceful shutdown. One background timer thread, a min-heap, lazy cancellation, `wait_until` with re-arm.

<br>

#### Skeleton

```cpp
// scheduler.h
#pragma once
#include <atomic>
#include <chrono>
#include <condition_variable>
#include <cstdint>
#include <functional>
#include <mutex>
#include <queue>
#include <thread>
#include <unordered_set>
#include <vector>

class Scheduler {
public:
    using Clock     = std::chrono::steady_clock;
    using TimerId   = std::uint64_t;
    using Callback  = std::function<void()>;

    Scheduler() {
        m_worker = std::thread([this]{ run(); });
    }
    ~Scheduler() {
        {
            std::lock_guard<std::mutex> lk(m_mtx);
            m_running = false;
        }
        m_cv.notify_all();
        if (m_worker.joinable()) m_worker.join();
    }
    Scheduler(const Scheduler&)            = delete;
    Scheduler& operator=(const Scheduler&) = delete;

    TimerId schedule(Callback cb, std::chrono::milliseconds delay) {
        return add(std::move(cb), delay, std::chrono::milliseconds{0});
    }
    TimerId scheduleRepeating(Callback cb, std::chrono::milliseconds interval) {
        return add(std::move(cb), interval, interval);
    }
    bool cancel(TimerId id) {
        std::lock_guard<std::mutex> lk(m_mtx);
        m_cancelled.insert(id);
        // No notify needed: the task is simply skipped when it surfaces.
        return true;
    }

private:
    struct Task {
        Clock::time_point         deadline;
        TimerId                   id;
        Callback                  cb;
        std::chrono::milliseconds interval;
    };
    struct Cmp {
        bool operator()(const Task& a, const Task& b) const {
            return a.deadline > b.deadline;   // min-heap on deadline
        }
    };

    TimerId add(Callback cb, std::chrono::milliseconds delay,
                std::chrono::milliseconds interval) {
        TimerId id = m_nextId.fetch_add(1, std::memory_order_relaxed);
        Task t{ Clock::now() + delay, id, std::move(cb), interval };
        {
            std::lock_guard<std::mutex> lk(m_mtx);
            m_heap.push(std::move(t));
        }
        m_cv.notify_one();   // re-arm: a sooner task may now be at the top
        return id;
    }

    void run() {
        std::unique_lock<std::mutex> lk(m_mtx);
        while (m_running) {
            if (m_heap.empty()) {
                m_cv.wait(lk);                       // nothing to do → sleep
            } else {
                auto next = m_heap.top().deadline;
                if (m_cv.wait_until(lk, next) == std::cv_status::timeout) {
                    fireDue(lk);
                }
                // else: notified → re-loop, recompute top
            }
        }
    }

    void fireDue(std::unique_lock<std::mutex>& lk) {
        auto now = Clock::now();
        std::vector<Task> due;
        while (!m_heap.empty() && m_heap.top().deadline <= now) {
            Task t = m_heap.top();
            m_heap.pop();
            if (m_cancelled.count(t.id)) { m_cancelled.erase(t.id); continue; }
            if (t.interval.count() > 0) {            // recurring → re-insert (fixed-rate)
                Task next = t;
                next.deadline = now + t.interval;
                m_heap.push(std::move(next));
            }
            due.push_back(std::move(t));
        }
        lk.unlock();
        for (auto& t : due) t.cb();                  // run callbacks without the lock
        lk.lock();
    }

    std::priority_queue<Task, std::vector<Task>, Cmp> m_heap;
    std::unordered_set<TimerId>                       m_cancelled;
    std::mutex                                        m_mtx;
    std::condition_variable                           m_cv;
    std::thread                                       m_worker;
    std::atomic<TimerId>                              m_nextId{1};
    bool                                              m_running = true;
};
```

<br>

#### Test driver

```cpp
// main.cpp
#include "scheduler.h"
#include <atomic>
#include <cassert>
#include <iostream>

using namespace std::chrono;

int main() {
    // 1. One-shot fires after the delay (and roughly on time)
    {
        Scheduler sch;
        std::atomic<bool> fired{false};
        auto t0 = steady_clock::now();
        sch.schedule([&]{ fired = true; }, milliseconds(50));
        std::this_thread::sleep_for(milliseconds(150));
        assert(fired.load());
        auto elapsed = duration_cast<milliseconds>(steady_clock::now() - t0).count();
        assert(elapsed >= 50);
        std::cout << "[1] one-shot fired OK\n";
    }

    // 2. Ordering: a later-scheduled but sooner-deadline task fires first (re-arm)
    {
        Scheduler sch;
        std::vector<int> order;
        std::mutex om;
        auto rec = [&](int v){ std::lock_guard<std::mutex> g(om); order.push_back(v); };
        sch.schedule([&]{ rec(200); }, milliseconds(200));
        sch.schedule([&]{ rec(50);  }, milliseconds(50));   // sooner → must fire first
        std::this_thread::sleep_for(milliseconds(300));
        assert(order.size() == 2);
        assert(order[0] == 50 && order[1] == 200);
        std::cout << "[2] deadline ordering / re-arm OK\n";
    }

    // 3. Cancellation: a cancelled one-shot never fires
    {
        Scheduler sch;
        std::atomic<bool> fired{false};
        auto id = sch.schedule([&]{ fired = true; }, milliseconds(100));
        assert(sch.cancel(id));
        std::this_thread::sleep_for(milliseconds(200));
        assert(!fired.load());
        std::cout << "[3] cancellation OK\n";
    }

    // 4. Recurring fires multiple times, then cancel stops it
    {
        Scheduler sch;
        std::atomic<int> count{0};
        auto id = sch.scheduleRepeating([&]{ count.fetch_add(1); }, milliseconds(40));
        std::this_thread::sleep_for(milliseconds(220));     // ~5 ticks
        sch.cancel(id);
        int afterCancel = count.load();
        assert(afterCancel >= 3);                            // fired several times
        std::this_thread::sleep_for(milliseconds(150));
        assert(count.load() == afterCancel);                 // no more after cancel
        std::cout << "[4] recurring + cancel OK (fired " << afterCancel << "x)\n";
    }

    // 5. Graceful shutdown: pending tasks don't crash on destruction
    {
        Scheduler sch;
        sch.schedule([]{ std::cout << "  (should NOT print — destroyed first)\n"; },
                     seconds(10));
        // scope ends → destructor joins cleanly even with a pending far-future task
        std::cout << "[5] graceful shutdown OK\n";
    }

    std::cout << "All assertions passed.\n";
    return 0;
}
```

<br>

#### Compile and run

```bash
g++ -std=c++17 -Wall -Wextra -Wpedantic -fsanitize=thread \
    -pthread -o day41 main.cpp && ./day41
```

(Run under ThreadSanitizer to catch races between the producer threads, the timer thread, and shutdown. Timing-based assertions use generous margins so they pass even on a loaded CI box.)

<br>

#### Expected output pattern

```
[1] one-shot fired OK
[2] deadline ordering / re-arm OK
[3] cancellation OK
[4] recurring + cancel OK (fired 5x)
[5] graceful shutdown OK
All assertions passed.
```

(The "should NOT print" line never appears — the 10-second task is abandoned at shutdown.) ThreadSanitizer reports no data races.

<br>

#### Bonus Challenges

1. **Thread-pool offload** — instead of running callbacks on the timer thread, enqueue them onto a worker pool (Day 13's pattern). Prove that a slow callback no longer delays other timers.

2. **Active cancellation cleanup** — when the cancelled set exceeds, say, 25% of the heap, rebuild the heap to physically drop cancelled tasks and reclaim memory. Measure the memory difference under heavy cancel churn.

3. **Fixed-rate vs fixed-delay** — add a flag so recurring tasks can choose `deadline += interval` (fixed-rate, drift-free) vs `deadline = now + interval` (fixed-delay). Demonstrate the drift difference with a deliberately slow callback.

4. **`scheduleAt(timePoint)`** — add an absolute-time API. What changes if you allow `system_clock` time points (wall-clock cron) vs `steady_clock`? Discuss DST / clock-jump hazards.

5. **Hashed timing wheel** — implement a single-level timing wheel (array of buckets + a ticking thread) and benchmark insert/cancel against the heap with 100k+ timers. Show the O(1) advantage.

6. **`cancel` returns accurate status** — make `cancel(id)` return `false` if the timer already fired or never existed (track live ids), instead of always `true`. What extra state does that require?

<br><br>

---
---

# PART 6: DEEP DIVE — COMMON QUESTIONS

---
---

<br>

<h2 style="color: #2980B9;">📘 41.8 Q&A</h2>

<br>

#### Q1: "Why a min-heap and not a sorted list or a `std::map`?"

We need O(1) access to the *earliest* deadline (to know how long to sleep) and cheap insertion. A binary min-heap gives O(1) peek-min and O(log n) insert/pop — ideal. A sorted vector has O(1) peek but O(n) insert; a balanced `std::map<deadline, task>` also works (O(log n) insert, O(1) `begin()`) and is a fine alternative, especially if you want ordered iteration. The heap is the textbook choice.

<br>

#### Q2: "Why `steady_clock` instead of `system_clock`?"

`system_clock` tracks wall-clock time and can jump backward or forward (NTP corrections, manual changes, DST). A delay measured against it could fire wildly early or late. `steady_clock` is monotonic — guaranteed to only move forward at a constant rate — which is what relative delays ("5 seconds from now") require. Use `system_clock` only for *absolute wall-clock* scheduling, and accept the jump hazards.

<br>

#### Q3: "Why `wait_until` and not `sleep_for`?"

`sleep_for` can't be interrupted, so a task scheduled sooner than the current sleep would fire late. `wait_until` on a condition variable can be woken early by `notify` (the re-arm), letting a newly added nearer task preempt the sleep, and lets shutdown wake the thread immediately. It also takes an absolute deadline, immune to skew from time spent locking/computing before sleeping.

<br>

#### Q4: "What is the re-arm pattern?"

When a new task is scheduled with a deadline sooner than the timer thread's current sleep target, the scheduler calls `cv.notify_one()`. The timer thread wakes from `wait_until`, re-reads the heap top (now the sooner task), and sleeps until that nearer deadline instead. Without it, the new task would only fire after the old, later deadline.

<br>

#### Q5: "Why is cancelling a heap entry hard, and how do you handle it?"

A binary heap supports removing the *minimum* in O(log n) but finding and removing an *arbitrary* element is O(n). The common fix is **lazy cancellation**: record the id in a cancelled set (O(1)) and skip the task when it surfaces at the top. The downside is cancelled tasks linger until their deadline; periodically rebuild the heap if the cancelled set grows large.

<br>

#### Q6: "Why run callbacks outside the lock?"

If a callback runs while the scheduler's mutex is held, it can't schedule new timers (it would deadlock on the same lock), and a slow callback blocks the whole timer thread. So the timer thread collects due tasks, **releases the lock**, runs the callbacks, then re-acquires. For heavy callbacks, offload them to a thread pool so the timer thread only triggers work.

<br>

#### Q7: "Fixed-rate vs fixed-delay recurring — what's the difference?"

**Fixed-delay** schedules the next run at `now + interval` (relative to when it *finished*), so slow callbacks push subsequent runs later — the schedule drifts. **Fixed-rate** schedules at `originalDeadline + interval`, keeping a steady cadence regardless of callback duration (it may even "catch up" by firing back-to-back if it fell behind). Heartbeats want fixed-rate; retry backoffs want fixed-delay.

<br>

#### Q8: "When would you switch from a heap to a hashed timing wheel?"

When you have very large numbers of timers (e.g., a timeout per connection in a server with hundreds of thousands of sockets). A timing wheel gives O(1) insert, cancel, and per-tick expiry by bucketing timers into a circular array indexed by `(now + delay) mod wheelSize`. The trade-off is tick-granularity precision instead of exact deadlines — perfect for timeouts/heartbeats. Hierarchical wheels extend the range (Linux kernel, Netty, Kafka use them).

<br><br>

---

## Reflection Questions

1. Why does the scheduler need O(1) access to the earliest deadline, and which structure provides it?
2. Why must the scheduler use `steady_clock` rather than `system_clock`?
3. Explain the re-arm pattern: what triggers it and what would break without it?
4. Why is arbitrary cancellation hard in a heap, and how does lazy cancellation solve it?
5. Why must callbacks run outside the scheduler's lock, and when should they be offloaded entirely?
6. What do hashed timing wheels optimize, and what precision do they trade away?

---

## Interview Questions

1. "Design a timer/scheduler supporting `schedule`, `scheduleRepeating`, and `cancel`."
2. "What data structure orders the pending timers, and why?"
3. "Why `steady_clock` instead of `system_clock` for delays?"
4. "How does the timer thread sleep precisely yet wake when a sooner task arrives?"
5. "How do you cancel a timer when removing an arbitrary heap element is O(n)?"
6. "Why can't you run callbacks while holding the scheduler's lock?"
7. "Differentiate fixed-rate from fixed-delay recurring tasks."
8. "How do you handle a callback that takes longer than the timer interval?"
9. "What is a hashed timing wheel and when would you use one over a heap?"
10. "How would you guarantee a clean shutdown with tasks still pending in the heap?"

---

**Next**: Day 42 — Weekly Review →
