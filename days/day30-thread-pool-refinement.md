# Day 30: Thread Pool Refinement

[← Back to Study Plan](../lld-study-plan.md) | [← Day 29](day29-thread-pool.md)

> **Time**: ~1.5-2 hours
> **Goal**: Yesterday's pool runs tasks FIFO and hopes nothing breaks. Today we make it **production-grade**: a **priority task queue** so urgent work jumps the line, **named threads** so a debugger/profiler shows you which worker is stuck, **exception propagation** through futures (and the subtle ways it goes wrong), and an intro to **work stealing** — the technique that lets per-thread queues scale without a central bottleneck. Build on the Day 29 pool: add priority and bullet-proof exception forwarding.

---
---

# PART 1: PRIORITY SCHEDULING

---
---

<br>

<h2 style="color: #2980B9;">📘 30.1 Why FIFO Isn't Enough</h2>

A plain `std::queue` is strictly first-in-first-out. That's fine until two task classes compete: a flood of low-priority background work (log flushing, cache warming) and a trickle of latency-critical work (a user request). With FIFO, the user request waits behind everything submitted before it.

A **priority queue** reorders by importance: high-priority tasks are dequeued first regardless of submission order.

```
FIFO queue:     [bg][bg][bg][USER][bg]   → USER waits for 3 bg tasks
Priority queue: [USER][bg][bg][bg][bg]   → USER runs next
                  ▲ floats to the front by priority
```

We swap `std::queue<std::function<void()>>` for a `std::priority_queue<PriorityTask>` where each task carries a priority value.

<br>

#### The structure

```cpp
enum class Priority : int { Low = 0, Normal = 1, High = 2, Critical = 3 };

struct PriorityTask {
    int                   priority;     // higher = runs sooner
    std::uint64_t         seq;          // tie-breaker: FIFO within a priority
    std::function<void()> fn;

    // priority_queue is a MAX-heap on operator<, so "less important" = smaller.
    bool operator<(const PriorityTask& other) const {
        if (priority != other.priority)
            return priority < other.priority;   // higher priority = "greater"
        return seq > other.seq;                 // lower seq = "greater" (earlier first)
    }
};
```

Two pitfalls to get right:

- **`std::priority_queue` is a max-heap.** Its `top()` is the *largest* element by `operator<`. So "more important" must compare as *greater*. We make higher `priority` compare greater (`priority < other.priority` returns true when `this` is *less* important).
- **Ties need a stable order.** Without a tie-breaker, equal-priority tasks come out in heap order — effectively random, and can **starve** an early task if newer equal-priority tasks keep arriving. A monotonically increasing `seq` makes equal priorities FIFO: earlier `seq` (smaller) must compare *greater*, hence `seq > other.seq`.

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 30.2 Starvation and Aging</h2>

Priority scheduling has a famous hazard: **starvation**. If high-priority tasks never stop arriving, low-priority tasks may *never* run. The seq tie-breaker only orders *within* a priority level; it does nothing across levels.

Mitigations:

| Technique | Idea | Cost |
|-----------|------|------|
| **Aging** | Periodically bump the effective priority of long-waiting tasks | Bookkeeping; needs a sweep |
| **Reserved capacity** | Dedicate K workers that always pull the lowest non-empty level | Wastes K workers under load |
| **Weighted fair queuing** | Serve levels in a ratio (e.g., 4 high : 1 low) | More complex dequeue logic |

For an interview-grade pool, mention aging and move on. For a real system, weighted fair queuing or a deadline scheduler is usually the right answer — pure priority is rarely correct in production because of starvation.

<br><br>

---
---

# PART 2: DEBUGGABILITY — NAMING THREADS

---
---

<br>

<h2 style="color: #2980B9;">📘 30.3 Why Name Threads?</h2>

When you attach `gdb`/`lldb`, open a core dump, or look at `top -H` / a profiler, threads show up by ID. A stuck pool is much easier to diagnose when the thread is called `pool-worker-3` instead of `0x7f2a...`. Naming is a debugging affordance, not a correctness feature — but it pays for itself the first time production hangs.

There's no portable C++ standard API for this; it's OS-specific:

```cpp
#include <thread>
#include <string>

#if defined(__linux__)
  #include <pthread.h>
  inline void setThreadName(const std::string& name) {
      // Linux limit: 15 chars + NUL.
      pthread_setname_np(pthread_self(), name.substr(0, 15).c_str());
  }
#elif defined(__APPLE__)
  #include <pthread.h>
  inline void setThreadName(const std::string& name) {
      // macOS sets the name of the *calling* thread only.
      pthread_setname_np(name.c_str());
  }
#else
  inline void setThreadName(const std::string&) { /* no-op fallback */ }
#endif
```

In the worker:

```cpp
m_workers.emplace_back([this, i] {
    setThreadName("pool-worker-" + std::to_string(i));
    workerLoop();
});
```

| Platform | Call | Notes |
|----------|------|-------|
| **Linux** | `pthread_setname_np(thread, name)` | Max 15 chars; visible in `/proc/<pid>/task/<tid>/comm`, `htop`, `gdb` |
| **macOS** | `pthread_setname_np(name)` | Names the *calling* thread only — must call from inside the worker |
| **Windows** | `SetThreadDescription` | Wide-string; or the legacy SEH exception hack |

<br><br>

---
---

# PART 3: EXCEPTION PROPAGATION

---
---

<br>

<h2 style="color: #2980B9;">📘 30.4 The Default Behaviour Is Already (Mostly) Right</h2>

Recall Day 29: tasks are wrapped in `std::packaged_task`. When the wrapped callable throws, `packaged_task::operator()` **catches** the exception and stores it in the future's shared state via `std::make_exception_ptr`. The worker thread is unharmed. The exception re-emerges at the caller's `future.get()`:

```cpp
auto f = pool.submit([] { throw std::runtime_error("boom"); return 0; });
try {
    f.get();                       // re-throws std::runtime_error("boom") HERE
} catch (const std::exception& e) {
    std::cout << "caught: " << e.what() << "\n";   // "caught: boom"
}
```

```
worker thread                          caller thread
─────────────                          ─────────────
(*task)()                              future<int> f
  └─ user code throws ──┐
  packaged_task catches │
  stores exception_ptr ─┼──► shared state ──► f.get()
  worker keeps running  │                      └─ rethrows here
```

This is one of the strongest reasons to route every task through `packaged_task` rather than a bare `std::function`. With a bare function, an escaping exception unwinds out of the worker loop and hits `std::terminate` (an exception leaving a thread's top-level function calls `terminate`). The pool would die.

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 30.5 Where Exception Propagation Actually Breaks</h2>

The mechanism is robust, but three real bugs bite people:

#### Bug 1: The future is discarded

```cpp
pool.submit([] { throw std::runtime_error("silent"); });   // future ignored!
```

The returned `future` is a temporary that's immediately destroyed. The exception is stored, but **nobody ever calls `get()`**, so it vanishes silently. The task "failed" and no one knows.

> Fix: either keep the future and `get()` it, or — for fire-and-forget tasks — wrap the task body in a `try/catch` that routes the error to a logger/error-handler, so failures are observable even when no one waits.

<br>

#### Bug 2: `std::async`-style blocking destructor confusion

A `std::future` from a thread pool does **not** block on destruction (unlike the special future returned by `std::async` with the default launch policy). Pool futures are backed by `packaged_task`, so dropping one just discards the result. Good for fire-and-forget, but combined with Bug 1 it means silent failures. Know the difference.

<br>

#### Bug 3: Throwing inside a worker's own machinery

If the *pool's* code (not the user task) throws — say `notify`/allocation inside `workerLoop` — that exception escapes the worker. Keep the worker loop `noexcept`-clean: do allocations and risky work inside the wrapped task, not in the loop scaffolding.

<br>

#### A fire-and-forget wrapper that doesn't lose errors

```cpp
template <typename F>
void submitDetached(F&& f, std::function<void(std::exception_ptr)> onError) {
    submit([fn = std::forward<F>(f), onError]() mutable {
        try {
            fn();
        } catch (...) {
            if (onError) onError(std::current_exception());
        }
    });   // future discarded deliberately — errors go to onError, not the void
}
```

<br><br>

---
---

# PART 4: WORK STEALING (INTRO)

---
---

<br>

<h2 style="color: #2980B9;">📘 30.6 The Central-Queue Bottleneck</h2>

The Day 29 pool has **one** queue guarded by **one** mutex. Every worker contends on that single lock for every pop. At low core counts this is fine; at 32+ cores, the mutex becomes a serialization point — workers spend more time fighting over the lock than running tasks.

**Work stealing** removes the central lock. Each worker owns a **local deque**:

```
Worker 0: [t][t][t]  ← pushes/pops its OWN end (LIFO, cache-hot)
Worker 1: [t][t]
Worker 2: []         ← empty! STEALS from another worker's FAR end (FIFO)
Worker 3: [t][t][t][t]
              └── worker 2 steals from here ──┘
```

The rules:

- A worker **pushes and pops its own deque from the same end** (the "bottom") — LIFO, which is cache-friendly because the most recently produced task is likely still hot in cache, and it keeps recursive task splits depth-first.
- An **idle** worker (empty local deque) becomes a **thief**: it pops from the **top** (the opposite end) of a *victim's* deque — FIFO from the victim's perspective, which tends to grab the oldest, largest sub-task and minimizes contention with the owner.
- Owner and thieves access **opposite ends**, so they rarely touch the same memory — the deque needs synchronization only at the boundary (a single shared element), which lock-free deques (the Chase-Lev deque) exploit.

<br>

#### Why it scales

| | Central queue | Work stealing |
|---|--------------|---------------|
| **Contention** | All N workers on one lock | Mostly local, lock-free; contention only on steals |
| **Cache locality** | Poor (tasks bounce between cores) | Good (LIFO keeps producer & consumer on one core) |
| **Load balancing** | Perfect, but serialized | Self-balancing: idle threads pull from busy ones |
| **Complexity** | Low | High (Chase-Lev deque, ABA, memory ordering) |

This is what powers Intel TBB, the C++ parallel STL, Java's `ForkJoinPool`, Rust's Rayon, and Go's scheduler. Building a correct lock-free Chase-Lev deque needs the atomics and memory-ordering knowledge of **Day 31** — so we only sketch it here. The takeaway for interviews: *central queue = simple but a scaling bottleneck; work stealing = local deques + steal-from-others, the standard answer for "how do real pools scale?"*

<br><br>

---
---

# PART 5: BUILD EXERCISE

---
---

<br>

<h2 style="color: #2980B9;">📘 30.7 Exercise: Priority + Exceptions + Named Workers</h2>

Extend the Day 29 pool: replace the FIFO queue with a stable priority queue, name the worker threads, and prove that exceptions propagate to `future.get()` while the pool stays alive.

<br>

#### Skeleton

```cpp
// priority_pool.h
#pragma once
#include <condition_variable>
#include <cstdint>
#include <functional>
#include <future>
#include <mutex>
#include <queue>
#include <stdexcept>
#include <string>
#include <thread>
#include <type_traits>
#include <utility>
#include <vector>

#if defined(__linux__) || defined(__APPLE__)
  #include <pthread.h>
#endif

inline void setThreadName(const std::string& name) {
#if defined(__linux__)
    pthread_setname_np(pthread_self(), name.substr(0, 15).c_str());
#elif defined(__APPLE__)
    pthread_setname_np(name.c_str());
#else
    (void)name;
#endif
}

enum class Priority : int { Low = 0, Normal = 1, High = 2, Critical = 3 };

class PriorityThreadPool {
    struct Task {
        int                   priority;
        std::uint64_t         seq;
        std::function<void()> fn;

        bool operator<(const Task& o) const {       // max-heap ordering
            if (priority != o.priority) return priority < o.priority;
            return seq > o.seq;                      // earlier seq runs first
        }
    };

public:
    explicit PriorityThreadPool(std::size_t n =
            std::max<std::size_t>(1, std::thread::hardware_concurrency())) {
        m_workers.reserve(n);
        for (std::size_t i = 0; i < n; ++i)
            m_workers.emplace_back([this, i] {
                setThreadName("pool-worker-" + std::to_string(i));
                workerLoop();
            });
    }

    ~PriorityThreadPool() {
        { std::lock_guard<std::mutex> lk(m_mutex); m_stop = true; }
        m_cv.notify_all();
        for (auto& w : m_workers) if (w.joinable()) w.join();
    }

    PriorityThreadPool(const PriorityThreadPool&)            = delete;
    PriorityThreadPool& operator=(const PriorityThreadPool&) = delete;

    template <typename F, typename... Args>
    auto submit(Priority prio, F&& f, Args&&... args)
        -> std::future<std::invoke_result_t<F, Args...>>
    {
        using R = std::invoke_result_t<F, Args...>;
        auto tp = std::make_shared<std::packaged_task<R()>>(
            std::bind(std::forward<F>(f), std::forward<Args>(args)...));
        auto fut = tp->get_future();
        {
            std::lock_guard<std::mutex> lk(m_mutex);
            if (m_stop) throw std::runtime_error("submit on stopped pool");
            m_tasks.push(Task{ static_cast<int>(prio), m_seq++, [tp]{ (*tp)(); } });
        }
        m_cv.notify_one();
        return fut;
    }

    // Convenience overload defaulting to Normal priority.
    template <typename F, typename... Args>
    auto submit(F&& f, Args&&... args) {
        return submit(Priority::Normal, std::forward<F>(f), std::forward<Args>(args)...);
    }

private:
    void workerLoop() {
        for (;;) {
            std::function<void()> fn;
            {
                std::unique_lock<std::mutex> lk(m_mutex);
                m_cv.wait(lk, [this] { return m_stop || !m_tasks.empty(); });
                if (m_stop && m_tasks.empty()) return;
                fn = std::move(const_cast<Task&>(m_tasks.top()).fn);  // see Q5
                m_tasks.pop();
            }
            fn();
        }
    }

    std::vector<std::thread>        m_workers;
    std::priority_queue<Task>       m_tasks;
    std::uint64_t                   m_seq = 0;
    mutable std::mutex              m_mutex;
    std::condition_variable         m_cv;
    bool                            m_stop = false;
};
```

<br>

#### Test driver

```cpp
// main.cpp
#include "priority_pool.h"
#include <atomic>
#include <cassert>
#include <iostream>
#include <mutex>
#include <vector>

int main() {
    std::cout << "=== 1. Higher priority runs earlier ===\n";
    {
        // Single worker so ordering is deterministic. Block it, queue a
        // mix of priorities, then release — they must drain by priority.
        PriorityThreadPool pool(1);
        std::mutex order_mtx;
        std::vector<int> order;
        std::promise<void> gate;
        std::shared_future<void> open = gate.get_future().share();

        // First task blocks the single worker until we open the gate.
        pool.submit(Priority::Normal, [open] { open.wait(); });

        for (int p : {0, 3, 1, 2, 0, 3}) {   // Low, Critical, Normal, High, Low, Critical
            pool.submit(static_cast<Priority>(p), [p, &order, &order_mtx] {
                std::lock_guard<std::mutex> lk(order_mtx);
                order.push_back(p);
            });
        }
        gate.set_value();                    // release the worker
        // give the worker time to drain by destroying the pool (drains all)
        // (scope end joins after draining)
        // We must read 'order' after drain, so move pool into inner scope:
        // -> handled by waiting on a final sentinel:
        pool.submit(Priority::Low, [&order, &order_mtx] {
            std::lock_guard<std::mutex> lk(order_mtx);
            (void)order;
        }).get();

        std::cout << "  drain order: ";
        for (int p : order) std::cout << p << " ";
        std::cout << "\n";
        // Critical(3) before High(2) before Normal(1) before Low(0):
        assert(order.front() == 3);
    }

    std::cout << "=== 2. Exception propagates to get(), pool survives ===\n";
    {
        PriorityThreadPool pool(2);
        auto bad = pool.submit([] { throw std::runtime_error("boom"); return 0; });
        bool caught = false;
        try { bad.get(); }
        catch (const std::exception& e) {
            caught = true;
            std::cout << "  caught: " << e.what() << "\n";
        }
        assert(caught);
        // Pool still works after a task threw:
        auto ok = pool.submit([] { return 7 * 6; });
        std::cout << "  pool still alive, 7*6 = " << ok.get() << "\n";
        assert(ok.get() == 42);
    }

    std::cout << "=== 3. Stress: many mixed-priority tasks all complete ===\n";
    {
        PriorityThreadPool pool(4);
        std::atomic<int> done{0};
        std::vector<std::future<void>> fs;
        for (int i = 0; i < 5000; ++i) {
            Priority p = static_cast<Priority>(i % 4);
            fs.push_back(pool.submit(p, [&done] { done.fetch_add(1); }));
        }
        for (auto& f : fs) f.get();
        std::cout << "  done = " << done.load() << "\n";
        assert(done.load() == 5000);
    }

    std::cout << "All assertions passed.\n";
    return 0;
}
```

<br>

#### Compile and run

```bash
g++ -std=c++17 -Wall -Wextra -Wpedantic -fsanitize=thread -pthread \
    -o day30 main.cpp && ./day30
```

<br>

#### Expected output

```
=== 1. Higher priority runs earlier ===
  drain order: 3 3 2 1 0 0
=== 2. Exception propagates to get(), pool survives ===
  caught: boom
  pool still alive, 7*6 = 42
=== 3. Stress: many mixed-priority tasks all complete ===
  done = 5000
All assertions passed.
```

<br>

#### Bonus Challenges

1. **Aging** — add a background sweeper thread that periodically raises the effective priority of tasks waiting longer than a threshold, eliminating starvation. (Note: `std::priority_queue` can't re-key in place — you'll need a different container, e.g. a hand-rolled heap or a multiset.)

2. **Cancellable futures** — return a token alongside the future; a cancelled task is skipped (not run) when popped.

3. **Per-priority metrics** — track wait time and run time per priority level; print histograms on shutdown to expose starvation.

4. **`shutdown()` vs `shutdownNow()`** — implement explicit drain and abort, returning the list of un-run tasks from `shutdownNow()`.

5. **Work-stealing prototype** — give each worker its own `std::deque` under a per-worker mutex (not yet lock-free). Workers pop locally; if empty, lock a random victim and steal from its far end. Measure throughput vs the central queue at high core counts.

6. **Two-level scheduling** — combine a small reserved set of workers for `Critical` tasks with a shared pool for the rest, so latency-critical work never queues behind a CPU-bound batch.

<br><br>

---
---

# PART 6: DEEP DIVE — COMMON QUESTIONS

---
---

<br>

<h2 style="color: #2980B9;">📘 30.8 Q&A</h2>

<br>

#### Q1: "Why does `std::priority_queue` need the comparator written 'backwards'?"

It's a **max-heap**: `top()` returns the *greatest* element by `operator<`. So the task you want to run *first* must compare as the *greatest*. Higher priority → greater, hence `priority < other.priority` returns true when `this` is the lower-priority one. It feels inverted because intuition says "high priority = comes first = small," but the heap's notion of 'first' is 'largest'.

<br>

#### Q2: "Why the `seq` tie-breaker — isn't priority enough?"

Equal-priority tasks would otherwise come out in arbitrary heap order, which is non-deterministic and can starve an early-submitted task when later equal-priority tasks keep arriving. The monotonic `seq` makes ties strictly FIFO, restoring fairness within a level (and making tests deterministic).

<br>

#### Q3: "Doesn't priority scheduling cause starvation?"

Yes — across priority *levels*. If `Critical` tasks never stop arriving, `Low` tasks never run. The `seq` tie-breaker only fixes ordering *within* a level. Real fixes: aging (bump waiting tasks' priority over time), reserved low-priority capacity, or weighted fair queuing. Pure strict priority is rarely correct in production for exactly this reason.

<br>

#### Q4: "Is thread naming portable?"

No. There's no standard C++ API. Linux: `pthread_setname_np(thread, name)` (15-char limit). macOS: `pthread_setname_np(name)` — and it only names the *calling* thread, so you must call it from inside the worker. Windows: `SetThreadDescription`. It's a debugging aid, never a correctness feature; wrap it behind one helper with a no-op fallback.

<br>

#### Q5: "Why the `const_cast` on `m_tasks.top()`?"

`std::priority_queue::top()` returns a `const&` because mutating an element could violate the heap invariant. But we only want to *move out* the `fn` of the element we're about to `pop()` — and popping removes it from the heap anyway, so the invariant doesn't matter. The `const_cast` lets us move instead of copy the `std::function`. It's safe *here* because `priority` and `seq` (the keys) are untouched and the element is removed immediately after. (Alternative: store `std::shared_ptr<Task>` so the const element is just a pointer you copy.)

<br>

#### Q6: "Exactly how does an exception get from the worker to `get()`?"

The task runs inside `std::packaged_task::operator()`. If the callable throws, `packaged_task` catches it, calls `std::make_exception_ptr`, and stores that in the future's shared state (instead of a value). The worker thread is unaffected and continues. When the caller invokes `future.get()`, the stored `exception_ptr` is re-thrown in the *caller's* thread via `std::rethrow_exception`.

<br>

#### Q7: "What's the silent-failure trap with exceptions?"

If you `submit()` a throwing task and discard the returned future, the exception is dutifully stored — but since no one ever calls `get()`, it's never observed. The task silently failed. Fixes: keep and `get()` the future, or use a fire-and-forget wrapper that `try/catch`es and routes errors to a logger/handler.

<br>

#### Q8: "How does work stealing avoid the central-lock bottleneck?"

Each worker has its *own* deque and works its own end (LIFO, cache-hot). Idle workers steal from the *opposite* end of a victim's deque (FIFO). Because owner and thief touch opposite ends, they almost never contend — synchronization is needed only at the single boundary element, which a lock-free Chase-Lev deque handles with atomics. There's no single global lock, so throughput scales with cores. This is how TBB, Rayon, Java's ForkJoinPool, and the C++ parallel STL scale.

<br><br>

---

## Reflection Questions

1. Why must a priority comparator for `std::priority_queue` make the most-urgent task compare as the *greatest*?
2. What problem does the `seq` tie-breaker solve, and what does it *not* solve?
3. Describe two concrete ways to prevent low-priority starvation.
4. Trace an exception from a throwing task to the caller's `get()`. Which component catches it, stores it, and re-throws it?
5. What is the silent-failure trap, and how do you make fire-and-forget tasks observable?
6. Why does a central task queue become a bottleneck at high core counts, and how does work stealing fix it?

---

## Interview Questions

1. "Add priority scheduling to a thread pool. Write the comparator and explain the heap ordering."
2. "Your priority queue starves low-priority tasks. What are your options?"
3. "Why is a stable tie-breaker needed in a priority task queue?"
4. "How does an exception thrown inside a pooled task reach the code that submitted it?"
5. "A teammate submits tasks and ignores the futures; failures vanish silently. Diagnose and fix."
6. "Why would you name your worker threads, and how do you do it portably?"
7. "Explain work stealing. Why do owner and thief use opposite ends of the deque?"
8. "Compare a central locked queue vs per-worker work-stealing deques: contention, locality, complexity."
9. "Why is moving the task out of `priority_queue::top()` safe despite `top()` being const?"
10. "Design a pool where latency-critical requests never queue behind a CPU-bound batch job."

---

**Next**: Day 31 — Atomics & Memory Ordering →
