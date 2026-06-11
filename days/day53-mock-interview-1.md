# Day 53: Mock Interview #1

[← Back to Study Plan](../lld-study-plan.md) | [← Day 52](day52-policy-based-design-concepts.md)

> **Time**: ~1.5-2 hours
> **Goal**: Run your first full timed LLD mock interview. Today's problem set: **Thread Pool** (Day 29, `day29-thread-pool.md`), **LRU Cache** (Day 38, `day38-lru-cache.md`), and **Logger** (Day 36, `day36-logger.md`). The worked example is the **Thread Pool**. The point of a mock is not to invent new theory — you've built all three before — but to rehearse the *performance*: budget time, design on paper before coding, talk aloud, state trade-offs, write compiling code under pressure, and handle follow-up "what-ifs." Treat this exactly like the real thing.

---
---

# PART 1: HOW TO RUN A TIMED MOCK

---
---

<br>

<h2 style="color: #2980B9;">📘 53.1 The 60-Minute Structure</h2>

A typical LLD round is ~60 minutes for one substantial problem (or two smaller ones). Use this clock and *enforce it with a timer*:

```
 0:00 ─ 0:05   Clarify requirements & constraints   (ASK, don't assume)
 0:05 ─ 0:20   Design on paper: API, data structures, concurrency model
 0:20 ─ 0:55   Implement the core, talking aloud
 0:55 ─ 1:00   Test mentally / walk a trace; discuss follow-ups
```

The single biggest mistake candidates make is coding at minute 1. **Spend a third of the time before you type.** A clear 15-minute design makes the 35-minute implementation almost mechanical; a skipped design makes the implementation a flailing rewrite.

<br>

<h2 style="color: #2980B9;">📘 53.2 The Rules You Impose on Yourself</h2>

| Rule | Why |
|------|-----|
| **Use a real timer.** | Pressure is the skill being trained. No untimed "practice." |
| **Design on paper/whiteboard first.** | Forces you to commit to an API and data model before details swamp you. |
| **Talk aloud the entire time.** | The interviewer scores your *reasoning*, not just the final code. Silence = no signal. |
| **State every trade-off as you make a decision.** | "I'll use a `condition_variable` rather than spin so idle workers don't burn CPU." |
| **Write code that would actually compile.** | No hand-waving `// ... locking here ...`. Pseudo-code is a yellow flag. |
| **Ask before assuming.** | Bounded or unbounded queue? Graceful or immediate shutdown? Each changes the design. |
| **Narrate your testing.** | Walk one happy-path trace and one edge case out loud at the end. |

<br>

<h2 style="color: #2980B9;">📘 53.3 The Clarifying Questions Checklist (first 5 minutes)</h2>

Before any design, pin down scope. Generic prompts hide the real constraints:

- **Functional**: What operations must it support? What's explicitly out of scope?
- **Scale**: How many items / tasks / requests? Single process or distributed?
- **Concurrency**: Multi-threaded? How many producers/consumers? Is the structure shared?
- **Lifecycle**: How is it created and destroyed? Graceful shutdown semantics?
- **Failure**: What happens on overload, on exception in a task, on a full queue?
- **Performance**: Latency vs throughput priority? Any hard limits?

Write the agreed constraints in a corner of the board. They are your contract and your defense when an interviewer later says "but what about X?"

<br><br>

---
---

# PART 2: WORKED EXAMPLE — THREAD POOL

---
---

<br>

<h2 style="color: #2980B9;">📘 53.4 Step 1 — Clarify (≈5 min)</h2>

Prompt: *"Design and implement a thread pool."* Questions you'd ask, and reasonable answers to assume if the interviewer defers:

| Question | Assumed answer |
|----------|----------------|
| Fixed-size or dynamic pool? | Fixed N worker threads, N given at construction. |
| Do tasks return values? | Yes — `submit` returns a `std::future<R>`. |
| Bounded or unbounded task queue? | Unbounded for v1; mention backpressure as a follow-up. |
| What on shutdown — drain or drop? | Graceful: finish queued tasks, then join. |
| Exception in a task? | Captured into the future via `packaged_task`; doesn't kill the worker. |

<br>

<h2 style="color: #2980B9;">📘 53.5 Step 2 — Design Decomposition (≈10 min, on paper)</h2>

```
        submit(f, args...)                         workers (N threads)
   ┌──────────────────────┐                  ┌─────────────────────────┐
   │ wrap in packaged_task│                  │  loop:                  │
   │ get future           │   enqueue        │   wait on cv until      │
   │ push closure ──────────────────────────►│    (queue nonempty ||   │
   │ notify_one           │   [mutex+queue]  │     stopping)           │
   └──────────────────────┘                  │   pop task              │
            │                                 │   unlock, run task      │
            ▼                                 └─────────────────────────┘
       std::future<R>  ◄── result/exception set by packaged_task
```

**Components:**
- A `std::vector<std::thread>` of N workers.
- A `std::queue<std::function<void()>>` of type-erased closures (note: type erasure — Day 51!).
- A `std::mutex` + `std::condition_variable` guarding the queue.
- A `bool stop_` flag for shutdown.

**API sketch:**

```cpp
class ThreadPool {
public:
    explicit ThreadPool(std::size_t n);
    ~ThreadPool();                                   // graceful: drain + join

    template <typename F, typename... Args>
    auto submit(F&& f, Args&&... args)
        -> std::future<std::invoke_result_t<F, Args...>>;

    ThreadPool(const ThreadPool&) = delete;          // non-copyable
    ThreadPool& operator=(const ThreadPool&) = delete;
};
```

State the trade-offs aloud here:
- *"Tasks are erased to `std::function<void()>` so the queue is homogeneous; the return value flows back through a `packaged_task`/`future`."*
- *"One mutex + cv is the simplest correct design. If profiling shows the single lock is a bottleneck, I'd move to per-worker queues with work-stealing — but that's premature now."*
- *"Graceful shutdown: set `stop_`, notify all, join. Workers keep draining until the queue is empty *and* `stop_` is set."*

<br>

<h2 style="color: #2980B9;">📘 53.6 Step 3 — Reference Implementation Skeleton (≈35 min)</h2>

```cpp
#include <condition_variable>
#include <functional>
#include <future>
#include <mutex>
#include <queue>
#include <thread>
#include <vector>

class ThreadPool {
    std::vector<std::thread>          workers_;
    std::queue<std::function<void()>> tasks_;
    std::mutex                        mtx_;
    std::condition_variable           cv_;
    bool                              stop_ = false;

public:
    explicit ThreadPool(std::size_t n) {
        for (std::size_t i = 0; i < n; ++i)
            workers_.emplace_back([this] { worker_loop(); });
    }

    ~ThreadPool() {
        {
            std::lock_guard<std::mutex> lk(mtx_);
            stop_ = true;
        }
        cv_.notify_all();
        for (auto& t : workers_)
            if (t.joinable()) t.join();
    }

    ThreadPool(const ThreadPool&)            = delete;
    ThreadPool& operator=(const ThreadPool&) = delete;

    template <typename F, typename... Args>
    auto submit(F&& f, Args&&... args)
        -> std::future<std::invoke_result_t<F, Args...>>
    {
        using R = std::invoke_result_t<F, Args...>;

        // packaged_task captures the callable + its future.
        auto task = std::make_shared<std::packaged_task<R()>>(
            [f = std::forward<F>(f),
             ... a = std::forward<Args>(args)]() mutable {     // C++20 pack capture
                return f(a...);
            });

        std::future<R> fut = task->get_future();
        {
            std::lock_guard<std::mutex> lk(mtx_);
            if (stop_) throw std::runtime_error("submit on stopped pool");
            tasks_.emplace([task] { (*task)(); });   // erase to void()
        }
        cv_.notify_one();
        return fut;
    }

private:
    void worker_loop() {
        for (;;) {
            std::function<void()> job;
            {
                std::unique_lock<std::mutex> lk(mtx_);
                cv_.wait(lk, [this] { return stop_ || !tasks_.empty(); });
                if (stop_ && tasks_.empty()) return;   // drained — exit
                job = std::move(tasks_.front());
                tasks_.pop();
            }                                          // unlock BEFORE running
            job();                                     // run outside the lock
        }
    }
};
```

Three details to call out aloud while writing — these are the scoring moments:

1. **Run the task outside the lock.** Holding `mtx_` while executing a task serializes everything and can deadlock if the task submits more work. Pop under the lock, release, *then* run.
2. **The wait predicate** `stop_ || !tasks_.empty()` correctly handles spurious wakeups and the shutdown race.
3. **Drain-then-exit**: `if (stop_ && tasks_.empty()) return;` — on shutdown, finish queued work first (graceful), don't drop it.

> If targeting C++17 (no template-pack lambda capture), bind the arguments with a small helper or `std::bind`; mention this version difference aloud.

<br>

<h2 style="color: #2980B9;">📘 53.7 Step 4 — Trade-offs to Articulate</h2>

| Decision | What I chose | Alternative | When I'd switch |
|----------|--------------|-------------|-----------------|
| Queue | Single shared `std::queue` + one mutex | Per-worker queues + work-stealing | Lock contention shows up under profiling |
| Task storage | `std::function<void()>` (type erasure) | Hand-rolled vtable for less allocation | Hot path, tiny tasks, allocation-bound |
| Return values | `packaged_task` → `future` | Callbacks / promises | Caller prefers continuation style |
| Shutdown | Graceful drain | Immediate abort (clear queue) | Caller wants fast cancel over completeness |
| Sizing | Fixed N | Dynamic (grow/shrink) | Bursty load with long idle periods |

This table *is* the interview. Saying "I picked the simple correct thing and here's exactly when I'd pay for something fancier" is the senior signal.

<br><br>

---
---

# PART 3: SELF-EVALUATION RUBRIC

---
---

<br>

<h2 style="color: #2980B9;">📘 53.8 Score Yourself After Each Problem</h2>

Rate 0–4 on each axis (0 = absent, 2 = adequate, 4 = strong). Be honest; the gaps are your study list.

| Axis | 0–1 | 2 | 3–4 | Thread Pool focus |
|------|-----|---|-----|-------------------|
| **Correctness** | Compiles? Races? | Works on happy path | Handles edge cases, no UB | Drain-on-shutdown; run outside lock |
| **API design** | Awkward, leaky | Reasonable | Idiomatic, hard to misuse | `submit` returns `future`; non-copyable |
| **Concurrency safety** | Data races | Locks present | Minimal critical sections, correct predicates | cv predicate; no work under lock |
| **Communication** | Silent | Narrates code | Explains *why*, states trade-offs proactively | Vocalize the 3 details in §53.6 |
| **Complexity analysis** | None | States big-O | States amortized + memory + contention | submit O(1); join O(N) |

**Target: ≥ 3 average to "pass."** Below 2 on concurrency or communication is the most common silent rejection in LLD rounds.

<br>

<h2 style="color: #2980B9;">📘 53.9 Time Budget Self-Check</h2>

After the mock, log where the time actually went:

```
Clarify     : target 5m   actual ___   (did you ask, or assume?)
Design      : target 10m  actual ___   (did you commit to an API first?)
Implement   : target 35m  actual ___   (did you rewrite mid-way?)
Test/follow : target 10m  actual ___   (did you trace an example?)
```

If "Implement" overran because you skipped "Design," that's the lesson — not "code faster."

<br><br>

---
---

# PART 4: FOLLOW-UP "WHAT-IF" QUESTIONS

---
---

<br>

<h2 style="color: #2980B9;">📘 53.10 Thread Pool Follow-ups</h2>

#### "What if a task throws?"

With `packaged_task`, the exception is stored in the shared state and re-thrown when the caller calls `future::get()`. The worker thread is unaffected — it catches nothing itself, the `packaged_task` does. If you used raw `std::function<void()>` without a future, an uncaught exception would call `std::terminate` — so wrap the body in try/catch. Always mention this distinction.

#### "How would you add backpressure / bound the queue?"

Add a `max_queue_size_`. In `submit`, if the queue is full, either (a) block the producer on a second condition variable until space frees up, or (b) reject with an error/`std::optional`. State the trade-off: blocking gives flow control but can stall producers; rejecting needs caller retry logic.

#### "The single mutex is a bottleneck at 64 cores. Now what?"

Move to **per-worker queues with work-stealing**: each worker has its own deque, pushes/pops from one end lock-free or with a local lock, and steals from other workers' opposite end when idle. This is what `tbb` and Go's scheduler do. Mention it reduces contention from O(threads) on one lock to near-zero in the common case.

#### "Graceful vs immediate shutdown?"

Graceful (what we built): set `stop_`, workers drain the queue then exit. Immediate: also clear `tasks_` so queued-but-unstarted work is dropped, and optionally signal a cancellation token to running tasks. Offer both via a parameter.

<br>

<h2 style="color: #2980B9;">📘 53.11 LRU Cache Follow-ups (problem 2)</h2>

Design recap: `unordered_map<Key, list<Node>::iterator>` + a `std::list<Node>` (intrinsic doubly-linked list); `get` and `put` are O(1) by splicing the touched node to the front and evicting from the back. (See Day 38, `day38-lru-cache.md`.)

#### "Make it thread-safe."

A single `std::mutex` around `get`/`put` is correct and simple — but note `get` *mutates* (moves the node to front), so even reads need the write lock; a plain `shared_mutex` won't help unless you separate "lookup" from "touch." For higher throughput, **shard** the cache by `hash(key) % N`, each shard with its own lock — eviction stays per-shard.

#### "Add TTL / expiry."

Store an expiry timestamp per node; on `get`, treat expired entries as misses and evict lazily. A background sweeper (or a min-heap by expiry) handles proactive eviction. Trade-off: lazy expiry is simpler but lets dead entries occupy capacity until touched.

#### "LRU vs LFU — when?"

LRU evicts least-*recently*-used (recency); LFU evicts least-*frequently*-used (popularity). LRU is simpler and adapts fast to changing working sets; LFU resists one-off scans polluting the cache but needs frequency counters and aging. Mention W-TinyLFU as the modern hybrid.

<br>

<h2 style="color: #2980B9;">📘 53.12 Logger Follow-ups (problem 3)</h2>

Design recap: levels (DEBUG..FATAL), pluggable **sinks** (console/file), a formatter, and — crucially — **asynchronous** logging via a background thread draining a queue so the hot path never blocks on I/O. (See Day 36, `day36-logger.md`.)

#### "Make logging non-blocking on the hot path."

Push formatted messages onto a concurrent queue; a dedicated consumer thread writes them to sinks. The producer pays only an enqueue. This *is* the producer/consumer pattern — same machinery as the thread pool. Trade-off: on crash, queued-but-unflushed messages may be lost; offer a `flush()` and flush-on-fatal.

#### "Multiple threads logging at once — interleaving?"

Each log call must produce one atomic write to the sink (a full line). With async logging the consumer serializes writes naturally. With synchronous logging, hold a per-sink mutex for the duration of one message, or format into a thread-local buffer and write once.

#### "How do you avoid the cost of `DEBUG` logs in production?"

Check the level *before* formatting: `if (level < threshold_) return;` — and use a macro so the argument expressions aren't even evaluated when disabled. State that string formatting, not the I/O, is often the hidden cost.

<br><br>

---

## Reflection Questions

1. Where did your time actually go versus the 5/10/35/10 budget? Which phase do you need to discipline?
2. In the thread pool, why must the task run *outside* the lock, and what concretely goes wrong if it doesn't?
3. Which scored you lower — correctness or communication? What will you change next mock?
4. For each of the three problems, what was the single most important clarifying question?
5. The LRU `get` mutates state. Why does that complicate "just use a reader-writer lock"?
6. What is the common thread (producer/consumer + a queue) linking the thread pool, the async logger, and a bounded cache?

---

## Interview Questions

1. "Implement a thread pool with `submit` returning a `std::future`."
2. "Why run a task outside the queue lock? Show the deadlock if you don't."
3. "How does an exception in a submitted task propagate to the caller?"
4. "Add backpressure to the thread pool's task queue. Block or reject?"
5. "The single queue mutex is a bottleneck. Describe work-stealing."
6. "Design an O(1) LRU cache. Why those two data structures?"
7. "Make the LRU cache thread-safe and high-throughput. Why sharding over one lock?"
8. "Design an asynchronous logger. How do you keep the hot path non-blocking?"
9. "How do you make `DEBUG`-level logging zero-cost when disabled?"
10. "Compare LRU and LFU eviction; when would you pick each?"

---

**Next**: Day 54 — Mock Interview #2 →
