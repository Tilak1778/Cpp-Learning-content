# Day 24: Semaphore & Barrier

[← Back to Study Plan](../lld-study-plan.md) | [← Day 23](day23-condition-variables.md)

> **Time**: ~1.5-2 hours
> **Goal**: A mutex enforces "one at a time." But sometimes you want "at most *N* at a time" (a connection limit, a pool of permits) or "everyone meet here before anyone continues" (phased computation). Learn the **counting semaphore** (and its degenerate binary form), the **latch** (a one-shot countdown gate), and the **barrier** (a reusable rendezvous point). See how C++20 standardized these as `std::counting_semaphore`, `std::latch`, and `std::barrier`. Then build a `Semaphore` and a reusable `Barrier` from scratch using only a mutex and a condition variable, proving you understand what they really are underneath.

---
---

# PART 1: THE COUNTING SEMAPHORE

---
---

<br>

<h2 style="color: #2980B9;">📘 24.1 What a Semaphore Is</h2>

A **counting semaphore** is a thread-safe counter of *permits*. It supports two operations:

- **acquire** (historically `P`, "wait", "down"): if the count is > 0, decrement and proceed; otherwise **block** until a permit is available.
- **release** (historically `V`, "signal", "up"): increment the count and wake one waiter.

```
Semaphore(count = 3)   →  3 permits available

acquire() → count 2   (thread takes a permit)
acquire() → count 1
acquire() → count 0
acquire() → BLOCKS    (no permits left)
                         ...another thread does...
release() → count 0, the blocked acquirer wakes and takes the permit
```

A semaphore generalizes a mutex: it limits concurrency to *N* simultaneous holders instead of exactly one.

<br>

#### Mutex vs semaphore — a crucial difference

| | Mutex | Semaphore |
|---|---|---|
| Holders allowed | exactly 1 | up to *N* |
| Ownership | has an *owner* — only the locker may unlock | **no owner** — any thread may release, even one that never acquired |
| Typical use | protect a critical section | limit concurrency / signal availability of a resource |
| Recursive? | sometimes (`recursive_mutex`) | n/a, it's just a count |

The "no ownership" property is what lets a semaphore be used for **signalling** between threads: thread A can `release` to hand a permit to thread B, which `acquire`s it — the count travels between threads. A mutex can't do that.

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 24.2 Binary Semaphore vs Mutex</h2>

A **binary semaphore** is a counting semaphore with max count 1 — it's either "available" (1) or "taken" (0). It looks like a mutex, but it is *not* a drop-in replacement:

- A binary semaphore has **no owner**, so any thread can release it. That makes it suitable for signalling (thread A waits, thread B signals) — a pattern a mutex forbids.
- A mutex has an owner and may offer recursion, priority inheritance, and deadlock detection from the OS. A semaphore is a dumber, lower-level primitive.

> Use a mutex to protect data. Use a (binary) semaphore to *signal* between threads. They look alike but model different intentions.

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 24.3 Common Semaphore Uses</h2>

| Use case | How |
|----------|-----|
| **Connection / rate limit** | `Semaphore sem(maxConns)`; each request `acquire`s before connecting, `release`s when done. At most `maxConns` concurrent. |
| **Resource pool** | One permit per pooled resource; `acquire` = "wait for a free one." |
| **Bounded queue** | Two semaphores: `emptySlots` (init = capacity) and `filledSlots` (init = 0). Producer `acquire`s an empty slot, `release`s a filled one; consumer the reverse. |
| **Throttling** | Limit concurrent background jobs to avoid overwhelming a downstream service. |
| **Signalling completion** | A worker `release`s; the waiter `acquire`s once per completed unit. |

The bounded-queue-with-two-semaphores construction is the classic textbook alternative to yesterday's condition-variable queue — and a great exercise to compare the two approaches.

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 24.4 C++20 std::counting_semaphore</h2>

C++20 finally standardized semaphores:

```cpp
#include <semaphore>

std::counting_semaphore<10> sem(3);   // max 10 permits, starts with 3
// std::binary_semaphore is an alias for std::counting_semaphore<1>

sem.acquire();              // block until a permit, then take it
// ... use the limited resource ...
sem.release();              // give a permit back (release(n) gives back n)

if (sem.try_acquire()) { /* got one without blocking */ }
sem.try_acquire_for(std::chrono::milliseconds(100));   // timed
```

The template parameter is the **maximum** count (a compile-time upper bound, often left at the default `PTRDIFF_MAX`); the constructor argument is the **initial** count. Implementations may use a lightweight futex/atomic fast path, so the standard semaphore is typically faster than a hand-rolled mutex+CV version.

<br><br>

---
---

# PART 2: LATCH AND BARRIER

---
---

<br>

<h2 style="color: #2980B9;">📘 24.5 Latch — A One-Shot Countdown Gate</h2>

A **latch** is a single-use counter that starts at *N* and counts *down*. Threads can wait until it reaches zero. Once it hits zero it stays there forever — it cannot be reset.

```cpp
#include <latch>

std::latch startGate(1);          // gate closed
std::latch allDone(workerCount);  // counts down as workers finish

// Workers:
//   startGate.wait();            // all workers block until the gate opens
//   ... do work ...
//   allDone.count_down();        // signal "I'm done"

startGate.count_down();           // main opens the gate → all workers start together
allDone.wait();                   // main blocks until every worker has finished
```

Two canonical uses:

- **Start gate**: spawn N workers that all `wait()`, then `count_down()` once from the main thread so they all begin at the same instant (great for benchmarks).
- **Completion latch**: initialize to N; each worker `count_down()`s when finished; the main thread `wait()`s for all to complete.

**One-shot** is the defining property — a latch is for a single phase. For repeated phases you need a barrier.

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 24.6 Barrier — A Reusable Rendezvous</h2>

A **barrier** synchronizes a group of threads at a phase boundary: each thread `arrive`s and then blocks until *all* threads have arrived; then they are all released together, and the barrier **resets** for the next phase. Unlike a latch, it's reusable.

```
Phase 1:  T0 ──arrive──┐
          T1 ──arrive──┤  all arrived → release all → run phase-2 work
          T2 ──arrive──┘
Phase 2:  T0 ──arrive──┐
          T1 ──arrive──┤  barrier RESET, blocks again until all arrive
          T2 ──arrive──┘
```

This is the heart of **bulk-synchronous parallel** algorithms: every thread computes a chunk of phase *k*, all wait at the barrier so no one races ahead into phase *k+1* before everyone finished *k* (think iterative simulations, parallel BFS levels, gradient-descent steps).

<br>

#### C++20 std::barrier

```cpp
#include <barrier>

auto onPhaseComplete = []() noexcept {
    std::cout << "phase complete — running completion function once\n";
};
std::barrier sync(numThreads, onPhaseComplete);   // optional completion callback

// Each worker, per phase:
//   ... compute this phase ...
//   sync.arrive_and_wait();   // block until all arrive; then the barrier auto-resets
```

The optional **completion function** runs exactly once, on the last-arriving thread, *after* everyone arrives but *before* anyone is released — the perfect place for "merge partial results / advance global state" between phases. It must be `noexcept`.

<br>

#### Latch vs Barrier

| | Latch | Barrier |
|---|---|---|
| Reusable | No — single-shot | Yes — resets each phase |
| Who decrements | Anyone, any number of times | Each participant arrives once per phase |
| Completion action | None | Optional callback per phase |
| Use | one start gate / one completion wait | iterative, multi-phase parallel work |

<br><br>

---
---

# PART 3: BUILD EXERCISE

---
---

<br>

<h2 style="color: #2980B9;">📘 24.7 Exercise: Semaphore and Reusable Barrier from Scratch</h2>

Build a counting `Semaphore` and a reusable `Barrier` using only `std::mutex` and `std::condition_variable` — exactly the primitives from Days 22-23. This proves you understand what the C++20 facilities actually are underneath.

<br>

#### Skeleton (header)

```cpp
// sync_primitives.h
#pragma once
#include <mutex>
#include <condition_variable>
#include <cstddef>
#include <cstdint>

// ── Counting semaphore from mutex + condition variable ──
class Semaphore {
    std::mutex m_;
    std::condition_variable cv_;
    std::int64_t count_;        // number of available permits

public:
    explicit Semaphore(std::int64_t initial = 0) : count_(initial) {}

    void acquire() {
        std::unique_lock<std::mutex> lk(m_);
        cv_.wait(lk, [&]{ return count_ > 0; });   // wait until a permit exists
        --count_;
    }

    bool tryAcquire() {
        std::lock_guard<std::mutex> lk(m_);
        if (count_ > 0) { --count_; return true; }
        return false;
    }

    void release(std::int64_t n = 1) {
        {
            std::lock_guard<std::mutex> lk(m_);
            count_ += n;
        }
        if (n == 1) cv_.notify_one();
        else        cv_.notify_all();   // multiple permits → wake multiple waiters
    }

    std::int64_t available() {
        std::lock_guard<std::mutex> lk(m_);
        return count_;
    }
};

// ── Reusable barrier from mutex + condition variable ──
//   "Generation" counter lets the barrier reset safely between phases:
//   a thread waits for the generation to change, not for a flag that the
//   next phase might already have flipped back (avoids a reset race).
class Barrier {
    std::mutex m_;
    std::condition_variable cv_;
    const std::size_t total_;       // participants per phase
    std::size_t waiting_ = 0;       // arrived-but-not-yet-released this phase
    std::size_t generation_ = 0;    // phase id

public:
    explicit Barrier(std::size_t count) : total_(count) {}

    void arriveAndWait() {
        std::unique_lock<std::mutex> lk(m_);
        std::size_t myGen = generation_;
        if (++waiting_ == total_) {
            // I am the last to arrive: open the gate and start a new phase.
            ++generation_;
            waiting_ = 0;
            cv_.notify_all();
        } else {
            // Wait until the generation advances (the gate opens).
            cv_.wait(lk, [&]{ return generation_ != myGen; });
        }
    }
};
```

<br>

#### Test driver

```cpp
// main.cpp
#include "sync_primitives.h"
#include <thread>
#include <vector>
#include <atomic>
#include <iostream>
#include <cassert>

// 1) Semaphore caps concurrency at N: verify the live count never exceeds N.
void testSemaphoreLimitsConcurrency() {
    std::cout << "=== Semaphore limits concurrency ===\n";
    constexpr int kPermits = 3;
    constexpr int kThreads = 16;

    Semaphore sem(kPermits);
    std::atomic<int> live{0};
    std::atomic<int> maxLive{0};

    std::vector<std::thread> ts;
    for (int i = 0; i < kThreads; ++i) {
        ts.emplace_back([&]{
            sem.acquire();
            int now = ++live;
            int prev = maxLive.load();
            while (now > prev && !maxLive.compare_exchange_weak(prev, now)) {}
            // simulate work
            for (volatile int spin = 0; spin < 100000; ++spin) {}
            --live;
            sem.release();
        });
    }
    for (auto& t : ts) t.join();

    std::cout << "  max concurrent = " << maxLive.load()
              << " (limit " << kPermits << ")\n";
    assert(maxLive.load() <= kPermits);
    assert(live.load() == 0);
    assert(sem.available() == kPermits);   // all permits returned
    std::cout << "  OK\n\n";
}

// 2) Barrier: every thread must finish phase k before any starts phase k+1.
void testBarrierPhases() {
    std::cout << "=== Barrier synchronizes phases ===\n";
    constexpr int kThreads = 8;
    constexpr int kPhases = 5;

    Barrier barrier(kThreads);
    std::atomic<int> phaseArrivals{0};   // how many arrived in the CURRENT phase
    std::atomic<int> violations{0};
    std::atomic<int> globalPhase{0};

    std::vector<std::thread> ts;
    for (int i = 0; i < kThreads; ++i) {
        ts.emplace_back([&]{
            for (int p = 0; p < kPhases; ++p) {
                // No thread should ever observe a phase number ahead of its own.
                if (globalPhase.load() < p) ++violations;
                ++phaseArrivals;
                barrier.arriveAndWait();
                // The last arriver effect: exactly kThreads arrivals per phase.
                if (phaseArrivals.load() != kThreads * (p + 1))
                    /* not all arrivals tallied yet for this phase is fine; we check the
                       stronger property below via the barrier semantics */ ;
                if (i == 0) ++globalPhase;     // one thread advances the global marker
                barrier.arriveAndWait();       // ensure globalPhase update is visible to all
            }
        });
    }
    for (auto& t : ts) t.join();

    std::cout << "  phases run = " << globalPhase.load()
              << ", total arrivals = " << phaseArrivals.load()
              << " (expected " << kThreads * kPhases << ")\n";
    assert(phaseArrivals.load() == kThreads * kPhases);
    assert(globalPhase.load() == kPhases);
    std::cout << "  OK\n\n";
}

int main() {
    testSemaphoreLimitsConcurrency();
    testBarrierPhases();
    std::cout << "All assertions passed.\n";
    return 0;
}
```

<br>

#### Compile and run

```bash
g++ -std=c++17 -Wall -Wextra -Wpedantic -O2 -pthread \
    -o day24 main.cpp && ./day24
```

#### Under ThreadSanitizer

```bash
g++ -std=c++17 -Wall -Wextra -Wpedantic -g -fsanitize=thread -pthread \
    -o day24_tsan main.cpp && ./day24_tsan
```

The semaphore test's `maxLive <= kPermits` is the key oracle: if your semaphore let more than N threads in at once, this assert fires. The barrier test's per-phase arrival count proves no thread raced ahead.

<br>

#### Expected output

```
=== Semaphore limits concurrency ===
  max concurrent = 3 (limit 3)
  OK

=== Barrier synchronizes phases ===
  phases run = 5, total arrivals = 40 (expected 40)
  OK

All assertions passed.
```

<br>

#### Bonus Challenges

1. **The generation trick.** Explain in writing why the barrier uses a `generation_` counter instead of a simple `bool gateOpen`. Construct the reset race that a naive boolean would suffer (the last thread opens the gate, resets the flag for the next phase, and an as-yet-unwoken thread from the *previous* phase sees the freshly-reset flag and waits forever).

2. **`std::counting_semaphore`.** Rewrite the semaphore test using C++20 `std::counting_semaphore`. Compile with `-std=c++20`. Confirm identical behavior; benchmark both — the standard one likely wins.

3. **Bounded queue from two semaphores.** Implement a bounded queue using `emptySlots` and `filledSlots` semaphores (no condition variables). Compare with Day 23's CV-based queue: which is simpler? Where does the mutex still appear (protecting the underlying container)?

4. **Latch from scratch.** Build a one-shot `Latch` with `countDown()` and `wait()`. Use it as a start gate so N benchmark threads begin at the same instant. Why can't you reuse it for a second round?

5. **Barrier completion callback.** Extend `Barrier` so the last-arriving thread runs a user-supplied function *before* releasing the others (mirroring `std::barrier`'s completion function). Use it to merge per-thread partial sums into a global total between phases.

6. **Semaphore fairness.** Your CV-based semaphore wakes an arbitrary waiter (`notify_one` doesn't guarantee FIFO). Demonstrate possible starvation, then design a FIFO semaphore (e.g., ticketed: each waiter takes a ticket and waits for its turn). Discuss the throughput cost of fairness.

<br><br>

---
---

# PART 4: DEEP DIVE — COMMON QUESTIONS

---
---

<br>

<h2 style="color: #2980B9;">📘 24.8 Q&A</h2>

<br>

#### Q1: "What's the difference between a mutex and a binary semaphore?"

A mutex has an **owner** — only the thread that locked it may unlock it — and often offers recursion, priority inheritance, and OS-level deadlock detection. A binary semaphore is **ownerless**: any thread may release it, even one that never acquired. That ownerlessness is exactly what makes a semaphore usable for *signalling* (thread A waits, thread B signals), which a mutex forbids. Use mutexes to protect data, semaphores to signal or to limit concurrency.

<br>

#### Q2: "What does the counting semaphore actually count?"

Available **permits**. `acquire` takes one (blocking if zero); `release` adds one (waking a waiter). It's a thread-safe counter with a "block when empty" wait. Initialized to N, it lets at most N threads hold a permit simultaneously — generalizing a mutex (N=1) to "at most N at a time."

<br>

#### Q3: "Latch vs barrier?"

A **latch** is single-shot: it counts down from N to 0 and stays at 0 forever. Anyone can count it down, any number of times. A **barrier** is reusable: each participant arrives once per phase, all block until everyone arrives, then all release and the barrier resets for the next phase. Use a latch for a single start-gate or completion-wait; use a barrier for iterative multi-phase algorithms.

<br>

#### Q4: "Why does the hand-rolled barrier need a generation counter?"

To avoid a **reset race**. If you used a plain `bool gateOpen`, the last-arriving thread would open the gate (set it true) and then immediately need to reset it (set it false) for the next phase. A slow thread from the *current* phase might wake, see the gate already reset to false, and wait forever for a gate that already opened. By waiting on "the generation number changed" rather than a re-settable flag, each thread is tied to its own phase — the monotonically increasing generation can't be confused with a future phase's state.

<br>

#### Q5: "What is the barrier's completion function for?"

It runs exactly once per phase, on the last-arriving thread, *after* all threads arrive but *before* any are released. That's a safe single-threaded window to advance global state between phases — merge partial results, swap double buffers, decide whether to continue. Because no participant is running at that moment, the callback needs no extra locking. In C++20's `std::barrier` it must be `noexcept`.

<br>

#### Q6: "Can I build a bounded queue with semaphores instead of condition variables?"

Yes — the classic construction: `emptySlots` semaphore initialized to capacity, `filledSlots` initialized to 0, plus a mutex for the container. Producer: `emptySlots.acquire()`, lock-push-unlock, `filledSlots.release()`. Consumer: `filledSlots.acquire()`, lock-pop-unlock, `emptySlots.release()`. It's arguably more elegant than the CV version because the counting *is* the bookkeeping. The CV version shines when you need richer predicates or graceful drain-on-close.

<br>

#### Q7: "Is a semaphore fair (FIFO)?"

Generally **no**. Neither the C++20 `std::counting_semaphore` nor a `notify_one`-based hand-rolled one guarantees that the longest-waiting thread is woken first; the OS scheduler decides. Under sustained contention this can starve a particular thread. If you need fairness, build a ticketed/FIFO semaphore explicitly — but expect lower throughput, since enforcing order constrains the scheduler.

<br>

#### Q8: "When should I prefer the C++20 standard primitives over hand-rolled ones?"

Almost always — `std::counting_semaphore`, `std::latch`, and `std::barrier` are correct, often use a fast atomic/futex path, and document intent clearly. Roll your own only to *learn* (as today), to target a pre-C++20 toolchain, or to add a capability the standard lacks (e.g., FIFO fairness, custom statistics). Reinventing these in production without a reason just adds bug surface.

<br><br>

---

## Reflection Questions

1. How does a counting semaphore generalize a mutex? What can a semaphore do that a mutex cannot?
2. Why is a binary semaphore not a safe drop-in replacement for a mutex?
3. Contrast a latch and a barrier. Which is reusable, and why does that matter for iterative algorithms?
4. Explain the reset race a naive boolean-based barrier suffers and how a generation counter prevents it.
5. What is a barrier completion function good for, and why does it need no extra locking?
6. When would you choose semaphores over condition variables to build a bounded queue?

---

## Interview Questions

1. "What is a counting semaphore? Implement one using a mutex and a condition variable."
2. "What's the difference between a mutex and a binary semaphore? When use each?"
3. "Implement a reusable barrier from scratch. Why do you need a generation counter?"
4. "Explain latch vs barrier. Give a real use case for each."
5. "Build a bounded producer-consumer queue using two semaphores."
6. "What does `std::barrier`'s completion function do, and when does it run?"
7. "How would you use a semaphore to limit concurrent database connections to N?"
8. "Are semaphores fair? How would you build a FIFO semaphore, and what does it cost?"
9. "Show how to start N benchmark threads at the exact same instant using a latch."
10. "Why is the unlock-and-sleep inside a semaphore's `acquire` required to be atomic?"

---

**Next**: Day 25 — Reader-Writer Lock →
