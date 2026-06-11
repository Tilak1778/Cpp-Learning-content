# Day 37: Logger — Async & Thread-Safe

[← Back to Study Plan](../lld-study-plan.md) | [← Day 36](day36-logger.md)

> **Time**: ~1.5-2 hours
> **Goal**: Yesterday's logger is correct but **synchronous** — every `log()` call formats and writes inline, so the calling thread pays the I/O cost and concurrent calls corrupt the output. Today you make it **thread-safe** and then **asynchronous**: a `log()` call merely pushes a record onto a queue and returns immediately; a dedicated **background thread** drains the queue, batches writes, and flushes periodically. You'll implement the **producer-consumer** pattern with a `condition_variable`, handle **graceful shutdown** (drain everything on destruction), and compare a locked queue against lock-free designs.

---
---

# PART 1: FROM SYNC TO ASYNC — WHY

---
---

<br>

<h2 style="color: #2980B9;">📘 37.1 The Problem With Synchronous Logging</h2>

In Day 36, `log()` did everything on the caller's thread:

```cpp
void log(Level l, std::string msg) {
    if (!enabled(l)) return;
    LogRecord r{ ... };
    m_sink->write(r);              // ← formatting + I/O on the CALLER'S thread
    if (l >= Level::Error) m_sink->flush();
}
```

Two problems appear under real load:

**1. Latency coupling.** A request-handling thread that logs is now blocked on a disk write (or worse, a network sink). A 2 ms disk hiccup becomes 2 ms of added request latency. Logging — a *diagnostic* concern — should never sit on the critical path.

**2. Interleaving / corruption.** With multiple threads, two `m_sink->write(r)` calls run concurrently. `std::ofstream::operator<<` is not atomic across the whole line, so you get:

```
2026-06-11 14:33:07.4822026-06-11 14:33:07.483 [INFO ] thread B msg [INFO ] thread A msg
```

Garbled, interleaved lines. Unusable.

<br>

The async design fixes both:

```
 Producer threads                    Bounded queue              Consumer thread
 ┌───────────┐  push(record)   ┌────────────────────────┐   pop  ┌──────────────┐
 │ thread A  │ ──────────────► │ [r0][r1][r2] ... [rN]   │ ─────► │ background   │
 │ thread B  │ ──────────────► │   (guarded by mutex)    │        │ flush thread │──► sink
 │ thread C  │ ──────────────► │                         │        │  batches     │
 └───────────┘                 └────────────────────────┘        └──────────────┘
   returns fast                 the ONLY shared mutable state      single writer →
   (just a push)                                                   no interleaving
```

The caller's job shrinks to "build a record, push it" (microseconds). One consumer owns the sink, so writes are serialized and never interleave — and the I/O cost is paid off the hot path.

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 37.2 Step 1: A Thread-Safe (Still Synchronous) Logger</h2>

Before going async, the *minimal* fix is a mutex around the write. This serializes lines (no corruption) but still blocks the caller on I/O.

```cpp
#include <mutex>

class SyncLogger {
    Level                     m_threshold;
    std::shared_ptr<ILogSink> m_sink;
    std::mutex                m_mutex;
public:
    void log(Level l, std::string msg) {
        if (l < m_threshold) return;                     // filter is lock-free (atomic read OK)
        LogRecord r{ l, std::move(msg), now(), tid() };
        std::lock_guard<std::mutex> lk(m_mutex);         // serialize sink access
        m_sink->write(r);
        if (l >= Level::Error) m_sink->flush();
    }
};
```

This is correct and sometimes *good enough* (low log volume). But the lock is held across the I/O, so under contention threads queue up behind the slowest write. The async design moves the I/O outside any lock held by producers.

> **Subtlety:** even `m_threshold` is shared. If `setLevel()` can be called concurrently, make it `std::atomic<Level>` so the filter read in `log()` is race-free. A torn read of a small enum is unlikely to matter, but atomics make it correct and free on common platforms.

<br><br>

---
---

# PART 2: THE PRODUCER-CONSUMER QUEUE

---
---

<br>

<h2 style="color: #2980B9;">📘 37.3 Anatomy of a Blocking Queue</h2>

The heart of the async logger is a **bounded blocking queue**. Its contract:

- **`push(record)`** — producers add a record. If the queue is full, either *block* (back-pressure) or *drop* (bounded loss). 
- **`pop()`** — the consumer waits until an item is available (or shutdown), then removes one (or a batch).
- **Shutdown** — wakes the consumer so it can drain and exit.

Three pieces of machinery:

```
   std::mutex                guards the queue + the flags
   std::condition_variable   "not empty"  → wakes the consumer
   std::condition_variable   "not full"   → wakes blocked producers (back-pressure)
```

```cpp
#include <condition_variable>
#include <deque>
#include <mutex>

template <typename T>
class BoundedQueue {
    std::deque<T>           m_q;
    std::size_t             m_cap;
    bool                    m_closed = false;
    std::mutex              m_mtx;
    std::condition_variable m_notEmpty;
    std::condition_variable m_notFull;

public:
    explicit BoundedQueue(std::size_t cap) : m_cap(cap) {}

    // Returns false if the queue was closed before the item could be pushed.
    bool push(T item) {
        std::unique_lock<std::mutex> lk(m_mtx);
        m_notFull.wait(lk, [&]{ return m_q.size() < m_cap || m_closed; });
        if (m_closed) return false;
        m_q.push_back(std::move(item));
        lk.unlock();              // unlock BEFORE notify → woken thread doesn't immediately block
        m_notEmpty.notify_one();
        return true;
    }

    // Drain up to maxBatch items into `out`. Returns false only when the queue
    // is closed AND empty (the consumer's signal to exit).
    bool popBatch(std::vector<T>& out, std::size_t maxBatch) {
        std::unique_lock<std::mutex> lk(m_mtx);
        m_notEmpty.wait(lk, [&]{ return !m_q.empty() || m_closed; });
        if (m_q.empty() && m_closed) return false;
        while (!m_q.empty() && out.size() < maxBatch)
            out.push_back(std::move(m_q.front())), m_q.pop_front();
        lk.unlock();
        m_notFull.notify_all();   // we freed space → wake blocked producers
        return true;
    }

    void close() {
        { std::lock_guard<std::mutex> lk(m_mtx); m_closed = true; }
        m_notEmpty.notify_all();  // wake consumer so it can drain+exit
        m_notFull.notify_all();   // wake any blocked producers
    }
};
```

<br>

#### Why predicate-based `wait`

```cpp
m_notEmpty.wait(lk, [&]{ return !m_q.empty() || m_closed; });
```

The lambda form is `while (!pred()) wait();` under the hood. It guards against **spurious wakeups** (a CV may wake with no notify) and the **lost-wakeup / missed-notify** race (a notify that arrives before you wait). Never write a bare `wait(lk)` without rechecking the condition — it's a classic bug.

<br>

#### Why unlock before notify

`notify_one()` while still holding the lock means the woken thread wakes, tries to acquire the mutex you still hold, and immediately blocks again ("hurry up and wait"). Unlocking first lets it proceed straight to the critical section. It's a throughput optimization, not a correctness requirement — but it's idiomatic.

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 37.4 Full-Queue Policy: Block, Drop, or Grow</h2>

What should `push()` do when the queue is full? This is a real product decision:

| Policy | Behavior | Trade-off |
|--------|----------|-----------|
| **Block** (back-pressure) | Producer waits for space | No data loss, but logging can stall the app — the very thing async tried to avoid |
| **Drop newest** | Discard the incoming record | App never stalls; you lose the most recent (often most relevant) logs |
| **Drop oldest** | Evict front, push new | App never stalls; keeps recent context, loses history |
| **Overflow to sync** | Caller writes directly when full | No loss, no unbounded memory, but reintroduces latency spikes |
| **Unbounded** | Never full | No stalls, but a slow sink → unbounded memory → OOM |

Production loggers usually let you **configure** this. A common default: bounded queue with **block** for correctness-critical systems, or **drop-with-a-counter** (record how many were dropped, log that count) for throughput-critical ones. The dropped-count line is essential — silent loss hides incidents.

<br><br>

---
---

# PART 3: THE ASYNC LOGGER

---
---

<br>

<h2 style="color: #2980B9;">📘 37.5 Assembling It</h2>

```cpp
#include <atomic>
#include <thread>

class AsyncLogger {
    std::shared_ptr<ILogSink>   m_sink;
    std::atomic<Level>          m_threshold;
    BoundedQueue<LogRecord>     m_queue;
    std::thread                 m_worker;
    std::chrono::milliseconds   m_flushEvery{200};

public:
    AsyncLogger(std::shared_ptr<ILogSink> sink, Level threshold, std::size_t cap)
        : m_sink(std::move(sink)), m_threshold(threshold), m_queue(cap) {
        m_worker = std::thread([this]{ run(); });        // start consumer
    }

    ~AsyncLogger() {
        m_queue.close();                                  // signal: no more input
        if (m_worker.joinable()) m_worker.join();         // wait for full drain
    }

    AsyncLogger(const AsyncLogger&)            = delete;
    AsyncLogger& operator=(const AsyncLogger&) = delete;

    void setLevel(Level l) { m_threshold.store(l, std::memory_order_relaxed); }
    bool enabled(Level l) const { return l >= m_threshold.load(std::memory_order_relaxed); }

    void log(Level l, std::string msg) {
        if (!enabled(l)) return;                          // cheap atomic read, no lock
        LogRecord r{ l, std::move(msg),
                     std::chrono::system_clock::now(),
                     std::this_thread::get_id() };
        m_queue.push(std::move(r));                       // returns fast
    }

private:
    void run() {
        std::vector<LogRecord> batch;
        batch.reserve(256);
        auto lastFlush = std::chrono::steady_clock::now();

        while (m_queue.popBatch(batch, /*maxBatch=*/256)) {
            for (const auto& r : batch) m_sink->write(r); // single-writer: no interleaving
            batch.clear();

            auto now = std::chrono::steady_clock::now();
            if (now - lastFlush >= m_flushEvery) {        // periodic flush
                m_sink->flush();
                lastFlush = now;
            }
        }
        m_sink->flush();   // final flush on shutdown — durability guarantee
    }
};
```

<br>

#### Why batching

Draining up to 256 records per loop turns N tiny writes into one tight loop and **one flush per interval** instead of one per record. The cost of acquiring the queue mutex and signaling is amortized over the whole batch. This is the difference between ~50k logs/sec and ~5M logs/sec.

<br>

#### The consumer loop, in words

1. Block until at least one record is available (or shutdown).
2. Move *all available* records (up to a cap) into a local batch — releasing the lock fast.
3. Write the batch to the sink (no lock held — producers keep pushing).
4. Flush if the flush interval elapsed.
5. When the queue is **closed and empty**, exit the loop and do a **final flush**.

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 37.6 Graceful Shutdown — The Hard Part</h2>

The destructor must guarantee: **every record already pushed is written before the program continues.** The sequence:

```
~AsyncLogger():
   1. m_queue.close()        → sets m_closed, wakes the consumer
   2. consumer drains the queue completely (popBatch returns true while items remain,
                                            even though closed, because !empty())
   3. popBatch returns false only when CLOSED && EMPTY → loop exits
   4. consumer does final flush()
   5. m_worker.join()        → destructor returns
```

The critical invariant lives in `popBatch`:

```cpp
if (m_q.empty() && m_closed) return false;   // exit ONLY when closed AND drained
```

If you instead exited as soon as `m_closed` is true, you'd drop everything still queued. **Close means "no new input," not "stop now."**

> **Static-destruction trap.** If the logger is a global/static, threads may still be logging during program teardown, and destruction order across translation units is unspecified. A `log()` call after the worker has joined would push onto a closed queue (handled — `push` returns false) but a `log()` racing the destructor is undefined. For global loggers, prefer an explicit `shutdown()` call at end of `main()`, or a Meyers singleton whose lifetime you control.

<br><br>

---
---

# PART 4: LOCKED VS LOCK-FREE

---
---

<br>

<h2 style="color: #2980B9;">📘 37.7 Do You Need Lock-Free?</h2>

Our queue uses a `mutex` + `condition_variable`. Is that a bottleneck? Usually **no** — the lock is held only long enough to push/move a pointer-sized record, on the order of tens of nanoseconds. The disk/sink is millions of times slower. For the overwhelming majority of systems, a well-batched locked queue is the right answer.

When lock-free *might* help:

| Concern | Locked MPSC queue | Lock-free queue |
|---------|-------------------|-----------------|
| **Throughput (typical loads)** | Excellent with batching | Marginally better |
| **Tail latency under heavy contention** | Producers can briefly block on the lock | More predictable |
| **Priority inversion / real-time** | A low-prio thread holding the lock can stall a high-prio one | Avoided |
| **Implementation risk** | Low — standard pattern | High — ABA, memory ordering, lifetime bugs |

<br>

#### Lock-free option: a ring buffer (SPSC / MPSC)

A single-producer-single-consumer ring buffer with two atomic indices is the classic lock-free logging queue (used by spdlog's async mode, the LMAX Disruptor, etc.):

```cpp
// SPSC ring (one producer, one consumer) — head/tail are atomics.
template <typename T, std::size_t N>
class SpscRing {
    std::array<T, N>         m_buf;
    std::atomic<std::size_t> m_head{0};   // written by consumer
    std::atomic<std::size_t> m_tail{0};   // written by producer
public:
    bool push(T v) {
        auto t = m_tail.load(std::memory_order_relaxed);
        auto next = (t + 1) % N;
        if (next == m_head.load(std::memory_order_acquire)) return false;  // full
        m_buf[t] = std::move(v);
        m_tail.store(next, std::memory_order_release);                     // publish
        return true;
    }
    bool pop(T& out) {
        auto h = m_head.load(std::memory_order_relaxed);
        if (h == m_tail.load(std::memory_order_acquire)) return false;     // empty
        out = std::move(m_buf[h]);
        m_head.store((h + 1) % N, std::memory_order_release);
        return true;
    }
};
```

The `release`/`acquire` pairing is what makes the published item *visible* to the consumer. With *multiple* producers you need either a per-thread ring (one SPSC ring per producer, consumer round-robins) or a true MPSC algorithm — which is where complexity (and bugs) explode.

> **Recommendation:** start with the locked + batched queue. It is correct, easy to reason about, and fast enough. Reach for lock-free only with a measured bottleneck and a willingness to test memory ordering carefully (TSan, stress tests). "Lock-free" is not a synonym for "faster" — it's a synonym for "no thread can be blocked by another holding a lock," which matters mainly for latency tails and real-time constraints.

<br><br>

---
---

# PART 5: BUILD EXERCISE

---
---

<br>

<h2 style="color: #2980B9;">📘 37.8 Exercise: An Async, Thread-Safe Logger</h2>

Build an `AsyncLogger` on top of a bounded blocking queue and a background consumer thread. Verify with multiple producer threads, assert no records are lost, and confirm graceful drain on destruction.

<br>

#### Skeleton

```cpp
// async_logger.h
#pragma once
#include <atomic>
#include <chrono>
#include <condition_variable>
#include <deque>
#include <iostream>
#include <memory>
#include <mutex>
#include <sstream>
#include <string>
#include <thread>
#include <vector>

enum class Level : int { Trace=0, Debug=1, Info=2, Warn=3, Error=4, Fatal=5, Off=6 };

struct LogRecord {
    Level                                 level;
    std::string                           message;
    std::chrono::system_clock::time_point time;
    std::thread::id                       threadId;
};

inline const char* levelName(Level l) {
    switch (l) {
        case Level::Trace: return "TRACE"; case Level::Debug: return "DEBUG";
        case Level::Info:  return "INFO "; case Level::Warn:  return "WARN ";
        case Level::Error: return "ERROR"; case Level::Fatal: return "FATAL";
        default:           return "?????";
    }
}

class ILogSink {
public:
    virtual ~ILogSink() = default;
    virtual void write(const LogRecord&) = 0;
    virtual void flush() {}
};

// ── Bounded blocking queue ───────────────────────────────
template <typename T>
class BoundedQueue {
    std::deque<T>           m_q;
    std::size_t             m_cap;
    bool                    m_closed = false;
    std::mutex              m_mtx;
    std::condition_variable m_notEmpty, m_notFull;
public:
    explicit BoundedQueue(std::size_t cap) : m_cap(cap) {}

    bool push(T item) {
        std::unique_lock<std::mutex> lk(m_mtx);
        m_notFull.wait(lk, [&]{ return m_q.size() < m_cap || m_closed; });
        if (m_closed) return false;
        m_q.push_back(std::move(item));
        lk.unlock();
        m_notEmpty.notify_one();
        return true;
    }

    bool popBatch(std::vector<T>& out, std::size_t maxBatch) {
        std::unique_lock<std::mutex> lk(m_mtx);
        m_notEmpty.wait(lk, [&]{ return !m_q.empty() || m_closed; });
        if (m_q.empty() && m_closed) return false;
        while (!m_q.empty() && out.size() < maxBatch) {
            out.push_back(std::move(m_q.front()));
            m_q.pop_front();
        }
        lk.unlock();
        m_notFull.notify_all();
        return true;
    }

    void close() {
        { std::lock_guard<std::mutex> lk(m_mtx); m_closed = true; }
        m_notEmpty.notify_all();
        m_notFull.notify_all();
    }
};

// ── Async logger ─────────────────────────────────────────
class AsyncLogger {
    std::shared_ptr<ILogSink> m_sink;
    std::atomic<Level>        m_threshold;
    BoundedQueue<LogRecord>   m_queue;
    std::thread               m_worker;
    std::chrono::milliseconds m_flushEvery{200};
public:
    AsyncLogger(std::shared_ptr<ILogSink> sink, Level t, std::size_t cap)
        : m_sink(std::move(sink)), m_threshold(t), m_queue(cap) {
        m_worker = std::thread([this]{ run(); });
    }
    ~AsyncLogger() {
        m_queue.close();
        if (m_worker.joinable()) m_worker.join();
    }
    AsyncLogger(const AsyncLogger&) = delete;
    AsyncLogger& operator=(const AsyncLogger&) = delete;

    void setLevel(Level l) { m_threshold.store(l, std::memory_order_relaxed); }
    bool enabled(Level l) const { return l >= m_threshold.load(std::memory_order_relaxed); }

    void log(Level l, std::string msg) {
        if (!enabled(l)) return;
        LogRecord r{ l, std::move(msg),
                     std::chrono::system_clock::now(),
                     std::this_thread::get_id() };
        m_queue.push(std::move(r));
    }
private:
    void run() {
        std::vector<LogRecord> batch;
        batch.reserve(256);
        auto lastFlush = std::chrono::steady_clock::now();
        while (m_queue.popBatch(batch, 256)) {
            for (const auto& r : batch) m_sink->write(r);
            batch.clear();
            auto now = std::chrono::steady_clock::now();
            if (now - lastFlush >= m_flushEvery) { m_sink->flush(); lastFlush = now; }
        }
        m_sink->flush();
    }
};
```

<br>

#### Counting sink (for assertable, race-free verification)

```cpp
// The consumer is single-threaded, so this sink needs NO locking.
#include <unordered_map>
class CountingSink : public ILogSink {
public:
    std::atomic<int>                  total{0};
    std::unordered_map<int,int>       perProducer;   // touched only by consumer thread
    void write(const LogRecord& r) override {
        total.fetch_add(1, std::memory_order_relaxed);
        // (parse producer id out of the message for the test below)
        int id = std::stoi(r.message.substr(r.message.find('#') + 1));
        perProducer[id]++;
    }
    void flush() override {}
};
```

<br>

#### Test driver

```cpp
// main.cpp
#include "async_logger.h"
#include <cassert>

int main() {
    constexpr int kProducers   = 8;
    constexpr int kPerProducer = 5000;

    auto sink = std::make_shared<CountingSink>();

    {
        AsyncLogger log(sink, Level::Trace, /*cap=*/1024);

        std::vector<std::thread> producers;
        for (int p = 0; p < kProducers; ++p) {
            producers.emplace_back([&log, p] {
                for (int i = 0; i < kPerProducer; ++i)
                    log.log(Level::Info, "producer#" + std::to_string(p));
            });
        }
        for (auto& t : producers) t.join();

        // Below-threshold logs must be dropped before the queue:
        log.setLevel(Level::Error);
        for (int i = 0; i < 1000; ++i) log.log(Level::Info, "producer#0");  // filtered out

    }   // ~AsyncLogger: close + drain + join happens HERE

    int expected = kProducers * kPerProducer;
    std::cout << "expected=" << expected
              << " written=" << sink->total.load() << "\n";
    assert(sink->total.load() == expected);   // no records lost across the drain

    for (int p = 0; p < kProducers; ++p) {
        assert(sink->perProducer[p] == kPerProducer);
    }

    std::cout << "All assertions passed (no records lost, graceful drain).\n";
    return 0;
}
```

<br>

#### Compile and run

```bash
g++ -std=c++17 -Wall -Wextra -Wpedantic -fsanitize=thread \
    -pthread -o day37 main.cpp && ./day37
```

(Use `-fsanitize=thread` here — ThreadSanitizer catches data races the address sanitizer can't. Run it both with `-fsanitize=address` and `-fsanitize=thread`; never both at once.)

<br>

#### Expected output pattern

```
expected=40000 written=40000
All assertions passed (no records lost, graceful drain).
```

The key result: **every** pushed record is written despite 8 concurrent producers and a destructor-triggered drain, and the 1000 below-threshold logs were filtered before ever touching the queue. ThreadSanitizer reports no races.

<br>

#### Bonus Challenges

1. **Drop policy + counter** — change `push` to drop (instead of block) when full, incrementing an atomic `m_dropped`. On shutdown, log a final "`N records dropped`" line. Verify under a deliberately slow sink.

2. **Periodic flush correctness** — make the consumer wake on a timeout even when no records arrive (use `popBatch` with a `wait_for`) so a low-traffic logger still flushes within the interval. Test with a 50 ms producer gap.

3. **Lock-free SPSC ring** — replace the locked queue with the `SpscRing` from §37.7 for a single producer. Benchmark throughput vs the locked queue with `perf` / timing.

4. **Per-thread rings (MPSC)** — give each producer its own SPSC ring registered with the logger; have the consumer round-robin across rings. Measure contention reduction vs one shared locked queue.

5. **Backpressure vs drop A/B** — build a slow sink (sleep 1 ms per write) and 16 fast producers. Compare end-to-end app throughput under block vs drop policies. Which keeps the app responsive? Which loses data?

6. **Crash durability** — `abort()` mid-run with and without the final `flush()` in `run()`. Show that the final flush is what saves the tail of the log.

<br><br>

---
---

# PART 6: DEEP DIVE — COMMON QUESTIONS

---
---

<br>

<h2 style="color: #2980B9;">📘 37.9 Q&A</h2>

<br>

#### Q1: "Why does async logging fix line interleaving?"

Because the queue funnels all records to a **single consumer thread** that owns the sink. With exactly one writer, two `write()` calls can never overlap, so lines never tear. The producers contend only on the queue's mutex (held for nanoseconds), not on the I/O.

<br>

#### Q2: "Why a `condition_variable` and not a busy-spin or `sleep`?"

A CV lets the consumer **block** (consuming zero CPU) until a producer signals "not empty," and wake promptly when work arrives. A busy-spin burns a core; a fixed `sleep` adds latency and still wastes cycles. The CV is the precise primitive for "wait until a condition becomes true."

<br>

#### Q3: "Explain the predicate in `wait`. Why not a bare `wait(lk)`?"

`wait(lk, pred)` is `while (!pred()) cv.wait(lk);`. The loop re-checks the condition after every wakeup, which defends against **spurious wakeups** (the OS may wake a waiter with no notify) and ensures correctness if a notify arrived between releasing the lock and waiting. A bare `wait(lk)` trusts that every wakeup means "ready," which is false.

<br>

#### Q4: "How does graceful shutdown guarantee no lost records?"

`close()` only sets a flag and wakes the consumer; it does **not** clear the queue. `popBatch` keeps returning batches while items remain (`!empty()` is checked before `closed`), and returns `false` to exit *only* when the queue is both closed **and** empty. The destructor then `join()`s the worker, so it can't return until the final flush completes. Close means "no new input," not "stop."

<br>

#### Q5: "What happens if the queue fills up?"

It depends on the configured policy (§37.4): block the producer (back-pressure, no loss, possible app stall), drop newest/oldest (no stall, bounded loss — always with a dropped-count), or overflow to a synchronous write. There's no universally right choice; it's a product decision between never losing logs and never stalling the app.

<br>

#### Q6: "Is lock-free always faster?"

No. Lock-free means *no thread is ever blocked waiting on a lock held by another* — it improves **tail latency** and avoids priority inversion, which matters in real-time/HFT contexts. For throughput on typical workloads, a **batched** locked queue is usually as fast and far simpler and safer. Reach for lock-free only against a measured bottleneck, and verify memory ordering with TSan.

<br>

#### Q7: "Should formatting happen on the producer or the consumer?"

Prefer the **consumer**: keep the producer's work minimal (build a small movable record, push). Formatting (timestamp string, level name, the full line) on the consumer keeps the hot path lean and concentrates CPU on one thread. The caveat: if the message captures *references* to mutable data, that data could change before the consumer formats it — so capture *values* into the record (which we do).

<br>

#### Q8: "Why make `m_threshold` atomic but not the message?"

`m_threshold` is read by every producer (`enabled()`) and may be written by `setLevel()` on another thread — concurrent read/write of shared data is a data race unless atomic. The message is *moved into* a record owned by one producer until it's pushed; after pushing, only the consumer touches it. There's no shared concurrent access to a single message, so no atomic is needed there.

<br><br>

---

## Reflection Questions

1. What two problems does async logging solve that a plain mutex-guarded sync logger does not?
2. Why does funneling through a single consumer thread eliminate line interleaving?
3. Walk through the exact destructor sequence that guarantees every queued record is written.
4. Why must `wait()` use a predicate, and what bugs does the bare form invite?
5. List the full-queue policies and the trade-off each makes between data loss and app stalling.
6. When is lock-free worth the complexity, and when is a batched locked queue the better engineering choice?

---

## Interview Questions

1. "Make a logger thread-safe and asynchronous. Walk me through the design."
2. "Implement a bounded blocking queue with `mutex` + `condition_variable`."
3. "Why must `condition_variable::wait` take a predicate?"
4. "How do you guarantee no log records are lost when the logger is destroyed?"
5. "What should happen when the log queue is full? Defend your default."
6. "Explain the producer-consumer pattern and how it applies here."
7. "Locked vs lock-free queue for logging — when would you choose each?"
8. "Why does batching matter for async-logger throughput?"
9. "Should log records be formatted on the producer or consumer thread? Why?"
10. "What goes wrong with a global/static async logger during program shutdown?"

---

**Next**: Day 38 — Design an LRU Cache →
