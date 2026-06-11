# Day 29: Thread Pool Design

[← Back to Study Plan](../lld-study-plan.md) | [← Day 27](day27-futures-promises.md)

> **Time**: ~1.5-2 hours
> **Goal**: A thread pool is a fixed set of **worker threads** that pull **tasks** from a shared **queue**, so you pay thread-creation cost once instead of per task. Learn how the worker loop, the task queue, and a `condition_variable` fit together; how `submit()` returns a `std::future` via `std::packaged_task`; and how to shut down **gracefully** without lost tasks or hangs. Build a generic `ThreadPool` with configurable thread count and `submit<F>(F&&) -> future<result_of<F>>`.

---
---

# PART 1: WHY A THREAD POOL EXISTS

---
---

<br>

<h2 style="color: #2980B9;">📘 29.1 The Cost of Threads</h2>

A `std::thread` is not free. Creating one is a system call (`pthread_create` / `CreateThread`) that allocates a stack (often 1-8 MB of virtual address space), registers the thread with the kernel scheduler, and sets up thread-local storage. On Linux this is on the order of **10-100 µs**; the destruction (join) costs again.

If your workload is "handle 10,000 short tasks," spawning 10,000 threads is catastrophic:

```
Naive: one thread per task
  task 1: create thread (50µs) → run (5µs)  → join (20µs)
  task 2: create thread (50µs) → run (5µs)  → join (20µs)
  ...
  10,000 tasks: ~750ms of pure thread overhead, plus
                10,000 simultaneous stacks = gigabytes of VM,
                massive scheduler thrash (oversubscription)
```

```
Thread pool: N workers (N ≈ hardware_concurrency)
  startup: create N threads once (N × 50µs)
  task i:  push to queue (≈0.1µs) → a free worker picks it up
  shutdown: join N threads once
  10,000 tasks: thread overhead is paid N times, not 10,000
```

This is exactly the **Object Pool pattern** (Day 13) applied to threads: pre-create an expensive resource, reuse it many times. The difference is that threads have an active **run loop**, so the pool is not a passive free-list — each worker is continuously *pulling* work.

<br>

#### When to reach for it

| Use case | Why a pool fits |
|----------|-----------------|
| **Web/RPC servers** | Many short request handlers; cap concurrency to core count |
| **Parallel batch jobs** | Map work across cores without manual thread management |
| **Background task runners** | Logging, metrics, async I/O completion |
| **Pipeline stages** | Each stage is a pool consuming from the previous stage's output |

The payoff: bounded, predictable concurrency (you never oversubscribe the CPU), amortized thread cost, and a clean `submit() → future` API that decouples *what to run* from *where it runs*.

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 29.2 How Many Threads?</h2>

The right count depends on whether tasks are **CPU-bound** or **I/O-bound**:

| Workload | Recommended threads | Reasoning |
|----------|---------------------|-----------|
| **CPU-bound** | `std::thread::hardware_concurrency()` | More threads than cores just causes context-switch overhead; cores are the real parallelism limit |
| **I/O-bound** | Higher (cores × (1 + wait/compute)) | Threads spend most time blocked, so extra threads keep cores busy |
| **Mixed** | Profile, or use separate pools | A blocked I/O task shouldn't starve CPU tasks |

A classic estimate (Little's Law flavour): `threads = cores × (1 + waitTime/computeTime)`. A task that waits 90% of the time wants ~10× the cores. But blindly setting a huge count invites memory blowup and contention — measure.

`hardware_concurrency()` may return `0` if it can't detect the count; always clamp to at least 1.

<br><br>

---
---

# PART 2: ANATOMY OF A THREAD POOL

---
---

<br>

<h2 style="color: #2980B9;">📘 29.3 The Three Moving Parts</h2>

```
        submit(task) ──┐
                       ▼
                ┌──────────────────────────────┐
                │  Task Queue (std::queue)       │
                │  ┌────┐┌────┐┌────┐            │  guarded by mutex
                │  │ t1 ││ t2 ││ t3 │            │  + condition_variable
                │  └────┘└────┘└────┘            │
                └──────────────────────────────┘
                   ▲        ▲        ▲
          notify_one() wakes a waiting worker
                   │        │        │
         ┌─────────┴─┐ ┌────┴────┐ ┌─┴────────┐
         │ Worker 0  │ │ Worker 1│ │ Worker 2 │   each runs workerLoop():
         │ wait→pop  │ │ wait→pop│ │ wait→pop │     while(!stop) { pop; run; }
         │ →run task │ │ →run    │ │ →run     │
         └───────────┘ └─────────┘ └──────────┘
```

The three parts and their jobs:

1. **Task queue** — a `std::queue<std::function<void()>>`. Type-erased so any callable fits.
2. **Synchronization** — a `std::mutex` protecting the queue, plus a `std::condition_variable` so idle workers *sleep* instead of busy-spinning.
3. **Worker threads** — each runs a loop: wait for work, pop a task, run it, repeat, until told to stop.

<br>

#### Why `std::function<void()>`?

Tasks have heterogeneous signatures (`int f()`, `void g(std::string)`, lambdas with captures). The queue must hold them uniformly, so we **erase the type** into `std::function<void()>`. The return value and arguments are captured *inside* the closure, leaving a uniform `void()` interface. The result is delivered out-of-band through a `std::future`.

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 29.4 The Worker Loop</h2>

This is the heart of the pool. Every worker runs the same loop:

```cpp
void workerLoop() {
    for (;;) {
        std::function<void()> task;
        {
            std::unique_lock<std::mutex> lock(m_mutex);
            // Sleep until there is work OR we are shutting down.
            m_cv.wait(lock, [this] { return m_stop || !m_tasks.empty(); });

            // Shutdown condition: stop requested AND no remaining work.
            if (m_stop && m_tasks.empty())
                return;

            task = std::move(m_tasks.front());
            m_tasks.pop();
        }                       // <-- release lock BEFORE running the task
        task();                 // run outside the lock so other workers can pop
    }
}
```

Three subtleties that make this correct:

- **The predicate in `wait`** (`m_stop || !m_tasks.empty()`) guards against *spurious wakeups* and *lost wakeups*. `condition_variable::wait(lock, pred)` re-checks the predicate in a loop, so a stray wakeup just re-sleeps.
- **Run the task outside the lock.** If you called `task()` while holding `m_mutex`, only one worker could run at a time — you'd have a fancy single-threaded queue. Releasing the lock first (the `{ }` scope ends) is what makes it parallel.
- **The shutdown check is `stop && empty`,** not just `stop`. This drains remaining tasks before exiting (graceful shutdown). A `stop`-only check would discard queued work.

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 29.5 submit() and packaged_task</h2>

`submit()` must (a) wrap the user's callable so its result can be retrieved later, and (b) return a handle to that future result. `std::packaged_task<R()>` does exactly this: it wraps a callable and exposes a `std::future<R>` via `get_future()`. When the task is invoked, the future becomes ready.

```cpp
template <typename F, typename... Args>
auto submit(F&& f, Args&&... args)
    -> std::future<std::invoke_result_t<F, Args...>>
{
    using R = std::invoke_result_t<F, Args...>;

    // Bind args into a zero-argument callable, then wrap in packaged_task.
    auto taskPtr = std::make_shared<std::packaged_task<R()>>(
        std::bind(std::forward<F>(f), std::forward<Args>(args)...));

    std::future<R> result = taskPtr->get_future();

    {
        std::lock_guard<std::mutex> lock(m_mutex);
        if (m_stop)
            throw std::runtime_error("submit on stopped ThreadPool");
        // Type-erase: push a void() that invokes the packaged_task.
        m_tasks.emplace([taskPtr]() { (*taskPtr)(); });
    }
    m_cv.notify_one();
    return result;
}
```

Why the `shared_ptr`? `std::packaged_task` is **move-only**, but `std::function` (pre-C++23) requires its target to be **copyable**. A lambda that *moves* a `packaged_task` into its capture is itself move-only and won't fit in `std::function`. Wrapping the task in a `shared_ptr` makes the capturing lambda copyable (it copies the pointer, not the task), so it slots into `std::function<void()>` cleanly. The `shared_ptr` also keeps the task alive until the worker runs it.

> **C++20 note**: `std::bind` is somewhat discouraged in favour of lambdas. An equivalent capture-by-lambda form is shown in §29.9. We use `std::bind` here because it composes cleanly with a variadic `Args...` pack.

<br>

```
submit(compute, 21)
   │
   ├─ packaged_task<int()> wrapping compute(21)  ──► future<int> ─────► caller
   │                                                                    .get()
   └─ shared_ptr → lambda [tp]{ (*tp)(); }  ──push──► queue ──► worker runs it
                                                              future becomes ready
```

<br><br>

---
---

# PART 3: GRACEFUL SHUTDOWN

---
---

<br>

<h2 style="color: #2980B9;">📘 29.6 Stopping Without Losing Work or Hanging</h2>

Shutdown is where naive pools break. The destructor must:

1. Set `m_stop = true` **under the lock** (so workers see a consistent value).
2. `notify_all()` to wake *every* sleeping worker — a `notify_one()` would leave others asleep forever.
3. `join()` every worker thread so the pool object outlives its threads.

```cpp
~ThreadPool() {
    {
        std::lock_guard<std::mutex> lock(m_mutex);
        m_stop = true;
    }
    m_cv.notify_all();          // wake ALL workers
    for (std::thread& w : m_workers)
        if (w.joinable())
            w.join();           // wait for each to finish draining
}
```

Because the worker loop exits only when `stop && tasks.empty()`, every already-queued task runs before shutdown completes. This is **graceful** (drain) shutdown.

<br>

#### Drain vs Abort

| Policy | Worker exit condition | Use when |
|--------|----------------------|----------|
| **Drain** (default) | `stop && empty` | You want all submitted work to finish |
| **Abort** | `stop` (ignore queue) | Process is dying; discard pending work |

For an abort policy, change the worker's exit check to just `if (m_stop) return;` and clear the queue under the lock. Most pools default to drain.

<br>

#### The "destructor must join" rule

If a `std::thread` is destroyed while still `joinable()`, the program **calls `std::terminate()`**. So the pool *must* join (or detach) every worker before the `m_workers` vector is destroyed. Joining is correct here — detaching would let workers touch a destroyed pool. This is the same RAII discipline as Day 5: the destructor owns cleanup.

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 29.7 The notify_one vs notify_all Trap</h2>

A subtle deadlock: suppose you `notify_one()` on shutdown but two workers are asleep. One wakes, sees `stop`, exits. The other never gets a notification and sleeps forever — `join()` on it hangs.

Rule of thumb:
- **`notify_one()`** when you add exactly one unit of work (one task → wake one worker).
- **`notify_all()`** for state changes that *every* waiter must observe (shutdown, "queue closed").

Also: always change the shared state (`m_stop`, push to queue) **before** notifying, and ideally while holding the lock or at least before releasing notification. Notifying without changing state is a lost wakeup waiting to happen.

<br><br>

---
---

# PART 4: BUILD EXERCISE

---
---

<br>

<h2 style="color: #2980B9;">📘 29.8 Exercise: A Generic Thread Pool</h2>

Build a `ThreadPool` with a configurable worker count and a `submit()` that returns a `std::future`. Verify correctness with assertions and a stress test, and run it under ThreadSanitizer to prove there are no data races.

<br>

#### Skeleton

```cpp
// thread_pool.h
#pragma once
#include <condition_variable>
#include <functional>
#include <future>
#include <mutex>
#include <queue>
#include <stdexcept>
#include <thread>
#include <type_traits>
#include <utility>
#include <vector>

class ThreadPool {
public:
    explicit ThreadPool(std::size_t threadCount =
                            std::max<std::size_t>(1, std::thread::hardware_concurrency())) {
        m_workers.reserve(threadCount);
        for (std::size_t i = 0; i < threadCount; ++i)
            m_workers.emplace_back([this] { workerLoop(); });
    }

    ~ThreadPool() {
        {
            std::lock_guard<std::mutex> lock(m_mutex);
            m_stop = true;
        }
        m_cv.notify_all();
        for (std::thread& w : m_workers)
            if (w.joinable())
                w.join();
    }

    ThreadPool(const ThreadPool&)            = delete;
    ThreadPool& operator=(const ThreadPool&) = delete;

    template <typename F, typename... Args>
    auto submit(F&& f, Args&&... args)
        -> std::future<std::invoke_result_t<F, Args...>>
    {
        using R = std::invoke_result_t<F, Args...>;

        auto taskPtr = std::make_shared<std::packaged_task<R()>>(
            std::bind(std::forward<F>(f), std::forward<Args>(args)...));

        std::future<R> result = taskPtr->get_future();
        {
            std::lock_guard<std::mutex> lock(m_mutex);
            if (m_stop)
                throw std::runtime_error("submit() on stopped ThreadPool");
            m_tasks.emplace([taskPtr] { (*taskPtr)(); });
        }
        m_cv.notify_one();
        return result;
    }

    std::size_t size() const { return m_workers.size(); }

    std::size_t pending() const {
        std::lock_guard<std::mutex> lock(m_mutex);
        return m_tasks.size();
    }

private:
    void workerLoop() {
        for (;;) {
            std::function<void()> task;
            {
                std::unique_lock<std::mutex> lock(m_mutex);
                m_cv.wait(lock, [this] { return m_stop || !m_tasks.empty(); });
                if (m_stop && m_tasks.empty())
                    return;
                task = std::move(m_tasks.front());
                m_tasks.pop();
            }
            task();
        }
    }

    std::vector<std::thread>          m_workers;
    std::queue<std::function<void()>> m_tasks;
    mutable std::mutex                m_mutex;
    std::condition_variable           m_cv;
    bool                              m_stop = false;
};
```

<br>

#### Test driver

```cpp
// main.cpp
#include "thread_pool.h"
#include <atomic>
#include <cassert>
#include <chrono>
#include <iostream>
#include <numeric>
#include <vector>

int main() {
    std::cout << "=== 1. Basic submit + future ===\n";
    {
        ThreadPool pool(4);
        std::future<int> f = pool.submit([](int a, int b) { return a + b; }, 20, 22);
        int v = f.get();
        std::cout << "  20 + 22 = " << v << "\n";
        assert(v == 42);
    }

    std::cout << "=== 2. Many tasks, sum the results ===\n";
    {
        ThreadPool pool(4);
        std::vector<std::future<int>> futures;
        for (int i = 1; i <= 100; ++i)
            futures.push_back(pool.submit([](int x) { return x * x; }, i));

        long long total = 0;
        for (auto& f : futures)
            total += f.get();
        // sum of squares 1..100 = 338350
        std::cout << "  sum of squares 1..100 = " << total << "\n";
        assert(total == 338350);
    }

    std::cout << "=== 3. void tasks + shared atomic counter ===\n";
    {
        ThreadPool pool(8);
        std::atomic<int> counter{0};
        std::vector<std::future<void>> futures;
        for (int i = 0; i < 10000; ++i)
            futures.push_back(pool.submit([&counter] { counter.fetch_add(1); }));
        for (auto& f : futures) f.get();
        std::cout << "  counter = " << counter.load() << "\n";
        assert(counter.load() == 10000);
    }

    std::cout << "=== 4. Graceful shutdown drains queued work ===\n";
    {
        std::atomic<int> done{0};
        {
            ThreadPool pool(2);
            for (int i = 0; i < 50; ++i)
                pool.submit([&done] {
                    std::this_thread::sleep_for(std::chrono::milliseconds(1));
                    done.fetch_add(1);
                });
            // destructor here drains all 50 before joining
        }
        std::cout << "  completed after shutdown = " << done.load() << "\n";
        assert(done.load() == 50);
    }

    std::cout << "All assertions passed.\n";
    return 0;
}
```

<br>

#### Compile and run

```bash
g++ -std=c++17 -Wall -Wextra -Wpedantic -fsanitize=thread -pthread \
    -o day29 main.cpp && ./day29
```

Also run under AddressSanitizer to catch use-after-free in shutdown:

```bash
g++ -std=c++17 -Wall -Wextra -Wpedantic -fsanitize=address -pthread \
    -o day29_asan main.cpp && ./day29_asan
```

<br>

#### Expected output

```
=== 1. Basic submit + future ===
  20 + 22 = 42
=== 2. Many tasks, sum the results ===
  sum of squares 1..100 = 338350
=== 3. void tasks + shared atomic counter ===
  counter = 10000
=== 4. Graceful shutdown drains queued work ===
  completed after shutdown = 50
All assertions passed.
```

(ThreadSanitizer should report **no** data races.)

<br>

#### Bonus Challenges

1. **Bounded queue with backpressure** — cap the queue size. When full, `submit()` should *block* the caller on a second condition variable until space frees up, instead of letting the queue grow unbounded.

2. **`wait_for_all()` / barrier** — add a method that blocks until the queue is empty *and* no worker is mid-task. Track an `m_activeTasks` counter and signal a CV when it reaches zero.

3. **Explicit `shutdown()` + `shutdownNow()`** — separate the drain and abort policies into named methods, and make the destructor call `shutdown()` only if neither was called.

4. **Task cancellation** — wrap submitted tasks so a returned token can mark them cancelled; workers skip cancelled tasks when popping.

5. **Per-pool statistics** — atomically track tasks submitted, completed, peak queue depth, and busy-thread count. Print a summary on destruction.

6. **Exception safety audit** — what happens if a submitted task `throw`s? (It does *not* crash the worker — the exception is stored in the future. Confirm this, then preview Day 30 where we exploit it.)

<br><br>

---
---

# PART 5: DEEP DIVE — COMMON QUESTIONS

---
---

<br>

<h2 style="color: #2980B9;">📘 29.9 Q&A</h2>

<br>

#### Q1: "Why `shared_ptr<packaged_task>` instead of moving the task into the queue lambda?"

`std::packaged_task` is move-only. A lambda that captures it by move becomes move-only too, and a pre-C++23 `std::function` requires a **copyable** target — so it won't compile. The `shared_ptr` indirection makes the capturing lambda copyable (copying a pointer, not the task). Alternatives: a custom move-only `function` type, C++23's `std::move_only_function`, or storing tasks as `std::packaged_task<void()>` directly in the queue.

The lambda-capture form (C++14+, no `std::bind`):

```cpp
template <typename F>
auto submit(F&& f) -> std::future<std::invoke_result_t<F>> {
    using R = std::invoke_result_t<F>;
    auto tp = std::make_shared<std::packaged_task<R()>>(std::forward<F>(f));
    auto fut = tp->get_future();
    { std::lock_guard<std::mutex> lk(m_mutex); m_tasks.emplace([tp]{ (*tp)(); }); }
    m_cv.notify_one();
    return fut;
}
```

<br>

#### Q2: "Why must the task run *outside* the lock?"

If `task()` ran while holding `m_mutex`, no other worker could pop the next task — the entire pool would serialize through the lock and run tasks one at a time. Releasing the lock before invoking the task is precisely what allows N workers to run N tasks concurrently. Holding locks across user code is also a deadlock risk if the task tries to `submit()` (which needs the same lock).

<br>

#### Q3: "What's the difference between `notify_one` and `notify_all` here?"

`notify_one` wakes a single waiting worker — correct for `submit()` because one task needs one worker. `notify_all` wakes everyone — required on **shutdown** so every sleeping worker observes `m_stop` and exits; a `notify_one` on shutdown would leave the others asleep, and `join()` would hang forever.

<br>

#### Q4: "Why does the worker check `m_stop && m_tasks.empty()` rather than just `m_stop`?"

That's the graceful-shutdown (drain) policy: workers keep popping until the queue is empty, so already-submitted tasks aren't lost. Checking only `m_stop` is an **abort** policy that discards pending work — sometimes desirable, but should be a deliberate choice, not an accident.

<br>

#### Q5: "What happens if a task throws an exception?"

The worker invokes the task through `std::packaged_task::operator()`, which **catches** the exception and stores it in the shared state. The worker thread keeps running. When the caller calls `future.get()`, the exception is **re-thrown** there. So exceptions propagate to the *submitter*, not the worker — the pool stays healthy. (If you stored a raw `std::function` and the task threw, that exception would escape the worker loop and call `std::terminate`. `packaged_task` is what saves us. More on this in Day 30.)

<br>

#### Q6: "How do I size the pool?"

For CPU-bound work, `hardware_concurrency()` (one worker per core). For I/O-bound work, more — roughly `cores × (1 + waitTime/computeTime)` — because threads spend most of their time blocked. Mixed workloads often warrant *two* pools so a blocked I/O task can't starve CPU tasks. Always clamp `hardware_concurrency()` to ≥ 1 (it may report 0).

<br>

#### Q7: "Is `pending()` reliable for load-balancing decisions?"

It's a *snapshot* — by the time the caller reads it, workers may have popped tasks. It's fine for metrics and rough heuristics but never for correctness logic ("if pending == 0 then done") — that's a TOCTOU race. For an authoritative "everything is finished" signal, track active tasks and use a condition variable (Bonus 2).

<br>

#### Q8: "Why is the pool non-copyable?"

It owns `std::thread`s and a `std::mutex`, neither of which is copyable, and copying running workers makes no sense. Deleting the copy operations makes the intent explicit. (It could be made *movable* with care, but the worker lambdas capture `this`, so a move would dangle those captures — leave it non-movable unless you indirect through a `shared_ptr` impl.)

<br><br>

---

## Reflection Questions

1. Why is creating one thread per task pathological for short tasks? What two costs dominate?
2. Walk through the worker loop. Why is the `wait` predicate `m_stop || !m_tasks.empty()` and not just `!m_tasks.empty()`?
3. Why must the task be invoked *after* releasing the mutex?
4. How does `submit()` deliver a result back to the caller without the queue knowing the return type?
5. What exactly makes the shutdown in the destructor "graceful," and how would you change it to "abort"?
6. Why `notify_all()` on shutdown but `notify_one()` on submit?

---

## Interview Questions

1. "Implement a thread pool with a `submit()` that returns a `std::future`."
2. "Why use `std::packaged_task` instead of `std::promise` directly in `submit`?"
3. "Why does `std::function` need a copyable target, and how do you store a move-only `packaged_task` in it?"
4. "Walk through the worker loop and explain each line's role in correctness."
5. "How do you shut down a thread pool without losing queued tasks? Without hanging?"
6. "What happens if a submitted task throws? Where does the exception surface?"
7. "How many threads should a pool have? Defend your answer for CPU- vs I/O-bound work."
8. "Why notify under (or around) the lock, and why `notify_all` vs `notify_one`?"
9. "How would you add backpressure when the task queue grows unbounded?"
10. "What goes wrong if a `std::thread` is destroyed while still joinable, and how does the pool avoid it?"

---

**Next**: Day 30 — Thread Pool Refinement →
