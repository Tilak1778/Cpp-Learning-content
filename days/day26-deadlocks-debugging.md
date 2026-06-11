# Day 26: Deadlocks & Debugging

[← Back to Study Plan](../lld-study-plan.md) | [← Day 25](day25-reader-writer-lock.md)

> **Time**: ~1.5-2 hours
> **Goal**: A deadlock is when two or more threads wait on each other forever and the program silently hangs. Learn the **four Coffman conditions** that *all* must hold for a deadlock, the canonical AB-BA lock-ordering deadlock, and the three robust fixes: a **global lock ordering**, a **lock hierarchy** that enforces it at runtime, and the deadlock-free multi-lock acquisition of `std::lock` / `std::scoped_lock`. Then learn to *detect* deadlocks and data races with **ThreadSanitizer**. Build an intentional deadlock, watch TSan flag it, and fix it three different ways.

---
---

# PART 1: WHAT A DEADLOCK IS

---
---

<br>

<h2 style="color: #2980B9;">📘 26.1 The Canonical AB-BA Deadlock</h2>

The simplest deadlock: two threads, two mutexes, locked in opposite orders.

```cpp
std::mutex a, b;

void thread1() {
    std::lock_guard la(a);   // 1: locks a
    std::lock_guard lb(b);   // 3: wants b — but thread2 holds it
}
void thread2() {
    std::lock_guard lb(b);   // 2: locks b
    std::lock_guard la(a);   // 4: wants a — but thread1 holds it
}
```

```
Thread 1 holds a, wants b ───►┐
                              │  (b held by thread 2)
                              ▼
                          ┌───────┐
                          │  b    │
                          └───────┘
                              ▲
                              │  (a held by thread 1)
Thread 2 holds b, wants a ────┘

  Circular wait → neither can proceed → frozen forever
```

Each thread holds one lock and waits for the other, which will never be released. The program doesn't crash — it just **hangs**, often only under specific timing, which is what makes deadlocks so insidious to reproduce and debug.

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 26.2 The Four Coffman Conditions</h2>

A deadlock can occur **only if all four** of these hold simultaneously. Break any one, and deadlock becomes impossible.

| # | Condition | Meaning | How to break it |
|---|-----------|---------|-----------------|
| 1 | **Mutual exclusion** | Resources can't be shared; only one holder at a time | Use lock-free / shared structures where possible |
| 2 | **Hold and wait** | A thread holds resources while waiting for more | Acquire *all* locks at once (e.g., `std::lock`), or hold none while waiting |
| 3 | **No preemption** | A held resource can't be forcibly taken away | Use `try_lock` with backoff: if you can't get the next lock, release what you hold and retry |
| 4 | **Circular wait** | A cycle of threads each waiting on the next | Impose a **global lock ordering** so cycles can't form |

The most practical lever in C++ is **#4 (circular wait)** — eliminate it with consistent lock ordering — and **#2 (hold and wait)** — eliminate it with `std::scoped_lock`/`std::lock`. The other two are usually structural givens.

<br>

> **Key takeaway:** you don't need to break all four — breaking *one* prevents deadlock. Good designs typically break circular wait (ordering) and/or hold-and-wait (atomic multi-lock).

<br><br>

---
---

# PART 2: THE FIXES

---
---

<br>

<h2 style="color: #2980B9;">📘 26.3 Fix 1: Global Lock Ordering</h2>

If every thread acquires locks in the **same total order**, no cycle can form (you can't have A-before-B *and* B-before-A). Assign every lock a rank and always lock low-to-high.

```cpp
// Both threads now lock a BEFORE b — no matter their logical task.
void thread1() { std::lock_guard la(a); std::lock_guard lb(b); /* ... */ }
void thread2() { std::lock_guard la(a); std::lock_guard lb(b); /* ... */ }
```

But what if the locks are chosen at runtime — e.g., transferring money between two accounts, each with its own mutex? Order by a stable, unique key (address, ID):

```cpp
void transfer(Account& from, Account& to, int amount) {
    // Always lock the lower-addressed account first → consistent global order.
    std::mutex* first  = &from.mtx;
    std::mutex* second = &to.mtx;
    if (first > second) std::swap(first, second);   // order by address

    std::lock_guard l1(*first);
    std::lock_guard l2(*second);
    from.balance -= amount;
    to.balance   += amount;
}
```

Without the ordering, `transfer(X, Y)` and `transfer(Y, X)` on two threads is the textbook AB-BA deadlock. (Note: a self-transfer where `&from == &to` would self-deadlock with `std::mutex`; guard against it.)

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 26.4 Fix 2: std::lock / std::scoped_lock</h2>

The standard library will acquire multiple mutexes **atomically and deadlock-free** for you, using an internal try-and-back-off algorithm. This breaks the *hold-and-wait* condition — it never holds one while blocking on another.

```cpp
// C++11 way: std::lock, then adopt into guards.
void transfer(Account& from, Account& to, int amount) {
    std::lock(from.mtx, to.mtx);                               // locks both, no deadlock possible
    std::lock_guard l1(from.mtx, std::adopt_lock);             // adopt = "already locked, just own it"
    std::lock_guard l2(to.mtx,   std::adopt_lock);
    from.balance -= amount; to.balance += amount;
}

// C++17 way: one line, same guarantee.
void transfer17(Account& from, Account& to, int amount) {
    std::scoped_lock lk(from.mtx, to.mtx);                     // deadlock-free multi-lock
    from.balance -= amount; to.balance += amount;
}
```

`std::scoped_lock` is the modern default for "I need these N mutexes together." It does the right thing regardless of the order you pass the arguments. This is the cleanest fix when you can name all the needed locks at one point.

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 26.5 Fix 3: Lock Hierarchy (Enforced Ordering)</h2>

A global ordering convention is only as good as everyone's discipline. A **lock hierarchy** *enforces* it at runtime: each mutex has a level, and a thread may only acquire a lock of strictly *lower* level than the lowest it currently holds. Violations throw immediately — turning a rare, timing-dependent hang into a loud, deterministic error at the offending line.

```cpp
class HierarchicalMutex {
    std::mutex m_;
    const unsigned long level_;
    unsigned long prevLevel_ = 0;
    static thread_local unsigned long t_currentLevel;   // lowest level this thread holds

public:
    explicit HierarchicalMutex(unsigned long level) : level_(level) {}

    void lock() {
        if (t_currentLevel <= level_)            // must descend to a strictly lower level
            throw std::logic_error("lock hierarchy violated");
        m_.lock();
        prevLevel_ = t_currentLevel;             // save to restore on unlock
        t_currentLevel = level_;
    }
    void unlock() {
        t_currentLevel = prevLevel_;             // restore previous level first
        m_.unlock();
    }
    bool try_lock() {
        if (t_currentLevel <= level_) throw std::logic_error("lock hierarchy violated");
        if (!m_.try_lock()) return false;
        prevLevel_ = t_currentLevel;
        t_currentLevel = level_;
        return true;
    }
};
thread_local unsigned long HierarchicalMutex::t_currentLevel = ~0UL;  // start at "infinity"
```

Now `HierarchicalMutex high(10000), low(5000);` — a thread may lock `high` then `low` (descending), but locking `low` then `high` throws *at the moment of the violation*, with a stack trace pointing right at the bug. It is `Lockable`, so it works with `std::lock_guard`/`std::scoped_lock`.

<br><br>

---
---

# PART 3: DETECTING DEADLOCKS & RACES

---
---

<br>

<h2 style="color: #2980B9;">📘 26.6 ThreadSanitizer (TSan)</h2>

**ThreadSanitizer** is a compiler instrumentation (in GCC and Clang) that detects data races and lock-ordering problems *at runtime*. You compile with `-fsanitize=thread`, run your program, and it reports violations with full stack traces for both conflicting accesses.

```bash
g++ -std=c++17 -g -fsanitize=thread -pthread -o app app.cpp
./app
```

TSan catches two distinct classes of bug:

- **Data races** — two threads touch the same memory unsynchronized, one writing. (This is its primary, most reliable function.)
- **Lock-order inversions** — TSan tracks the order in which each thread acquires locks and warns when thread A locks `m1→m2` while thread B locks `m2→m1`, the AB-BA pattern. It can warn about a *potential* deadlock **even if the actual deadlock didn't occur on that run**, because it saw the inconsistent ordering.

That last point is huge: a deadlock might manifest once in a million runs, but TSan flags the dangerous *ordering* the first time it sees both orders, regardless of timing.

<br>

#### Costs and limits

| | |
|---|---|
| Slowdown | ~5-15× — for testing, not production |
| Memory | ~5-10× |
| Coverage | Only code paths actually executed (it's dynamic, not static) |
| Caveat | Can't combine with `-fsanitize=address`; run them separately |

The discipline: **run your test suite under TSan in CI.** Races that "never happen" on your machine surface immediately under instrumentation.

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 26.7 Other Detection Tools & Techniques</h2>

- **Deadlock-already-happened?** Attach `gdb`/`lldb` to the hung process, `thread apply all bt` — you'll see each thread parked inside `pthread_mutex_lock`, and the cycle is visible from which mutexes appear in the backtraces.
- **Helgrind / DRD** (Valgrind tools) — detect races and lock-order errors without recompiling (slower than TSan, no rebuild needed).
- **Timeouts as a smoke test** — `try_lock_for` with a timeout that logs "couldn't acquire in N ms" turns silent hangs into visible diagnostics.
- **Lock hierarchy in debug builds** — ship the `HierarchicalMutex` in debug builds to catch ordering bugs deterministically before they reach production.

<br><br>

---
---

# PART 4: BUILD EXERCISE

---
---

<br>

<h2 style="color: #2980B9;">📘 26.8 Exercise: Create, Detect, and Fix a Deadlock</h2>

Build a money-transfer system with a per-account mutex. First write the *buggy* version that deadlocks under concurrent opposite transfers, detect it with ThreadSanitizer (and observe the hang), then fix it three ways and verify correctness.

<br>

#### Skeleton (header) — both buggy and fixed transfer functions

```cpp
// bank.h
#pragma once
#include <mutex>
#include <vector>
#include <algorithm>
#include <cstddef>

struct Account {
    std::mutex mtx;
    long long balance = 0;
};

class Bank {
    std::vector<Account> accounts_;
public:
    explicit Bank(std::size_t n, long long initial) : accounts_(n) {
        for (auto& a : accounts_) a.balance = initial;
    }
    Account& at(std::size_t i) { return accounts_[i]; }
    std::size_t count() const { return accounts_.size(); }

    long long total() {
        // Lock everything in index order (a consistent global order) for a clean snapshot.
        std::vector<std::unique_lock<std::mutex>> locks;
        for (auto& a : accounts_) locks.emplace_back(a.mtx);
        long long sum = 0;
        for (auto& a : accounts_) sum += a.balance;
        return sum;
    }

    // ── BUGGY: locks `from` then `to` → AB-BA deadlock with the reverse transfer ──
    void transferBuggy(std::size_t from, std::size_t to, long long amount) {
        std::lock_guard<std::mutex> l1(accounts_[from].mtx);
        // (a sleep here would make the deadlock near-certain; even without it, it hits under load)
        std::lock_guard<std::mutex> l2(accounts_[to].mtx);
        accounts_[from].balance -= amount;
        accounts_[to].balance   += amount;
    }

    // ── FIX 1: global ordering by index ──
    void transferOrdered(std::size_t from, std::size_t to, long long amount) {
        if (from == to) return;
        std::size_t lo = std::min(from, to), hi = std::max(from, to);
        std::lock_guard<std::mutex> l1(accounts_[lo].mtx);   // always lower index first
        std::lock_guard<std::mutex> l2(accounts_[hi].mtx);
        accounts_[from].balance -= amount;
        accounts_[to].balance   += amount;
    }

    // ── FIX 2: std::scoped_lock (deadlock-free multi-lock) ──
    void transferScoped(std::size_t from, std::size_t to, long long amount) {
        if (from == to) return;
        std::scoped_lock lk(accounts_[from].mtx, accounts_[to].mtx);  // any order is safe
        accounts_[from].balance -= amount;
        accounts_[to].balance   += amount;
    }
};
```

<br>

#### Test driver

```cpp
// main.cpp
#include "bank.h"
#include <thread>
#include <vector>
#include <iostream>
#include <cassert>

// Run many concurrent transfers, half in each direction, then verify the
// total is conserved (money is neither created nor destroyed).
template <typename TransferFn>
bool runWorkload(const char* label, TransferFn transfer) {
    constexpr std::size_t kAccounts = 8;
    constexpr long long   kInitial  = 1000000;
    constexpr int         kThreads  = 8;
    constexpr int         kOps      = 100000;

    Bank bank(kAccounts, kInitial);
    long long startTotal = bank.total();

    std::vector<std::thread> ts;
    for (int t = 0; t < kThreads; ++t) {
        ts.emplace_back([&, t]{
            for (int i = 0; i < kOps; ++i) {
                std::size_t x = (t + i) % kAccounts;
                std::size_t y = (t + i + 1) % kAccounts;
                // Half the threads transfer x→y, half y→x: the AB-BA recipe.
                if (t % 2 == 0) transfer(bank, x, y, 1);
                else            transfer(bank, y, x, 1);
            }
        });
    }
    for (auto& t : ts) t.join();

    long long endTotal = bank.total();
    std::cout << "  [" << label << "] start total=" << startTotal
              << " end total=" << endTotal << "\n";
    return startTotal == endTotal;
}

int main() {
    std::cout << "=== Fix 1: global lock ordering ===\n";
    bool ok1 = runWorkload("ordered",
        [](Bank& b, std::size_t f, std::size_t t, long long a){ b.transferOrdered(f, t, a); });
    assert(ok1);

    std::cout << "=== Fix 2: std::scoped_lock ===\n";
    bool ok2 = runWorkload("scoped",
        [](Bank& b, std::size_t f, std::size_t t, long long a){ b.transferScoped(f, t, a); });
    assert(ok2);

    // To SEE the deadlock, uncomment the line below and run. It will hang
    // (and TSan will report a lock-order inversion). DO NOT leave it enabled.
    // std::cout << "=== Buggy (will hang!) ===\n";
    // runWorkload("buggy",
    //     [](Bank& b, std::size_t f, std::size_t t, long long a){ b.transferBuggy(f, t, a); });

    std::cout << "All assertions passed (no money created or destroyed).\n";
    return 0;
}
```

<br>

#### Compile and run the fixes

```bash
g++ -std=c++17 -Wall -Wextra -Wpedantic -O2 -pthread \
    -o day26 main.cpp && ./day26
```

#### Detect the deadlock / ordering bug with ThreadSanitizer

Uncomment the buggy block, then:

```bash
g++ -std=c++17 -Wall -Wextra -Wpedantic -g -fsanitize=thread -pthread \
    -o day26_tsan main.cpp && ./day26_tsan
```

TSan reports a **lock-order-inversion (potential deadlock)** warning naming both mutexes and the two stack traces that acquired them in opposite orders — even if the process hadn't fully frozen yet. The fixed versions produce no TSan warnings.

<br>

#### Expected output (fixes only)

```
=== Fix 1: global lock ordering ===
  [ordered] start total=8000000 end total=8000000
=== Fix 2: std::scoped_lock ===
  [scoped] start total=8000000 end total=8000000
All assertions passed (no money created or destroyed).
```

<br>

#### Bonus Challenges

1. **Watch it hang.** Enable the buggy workload (and add a `std::this_thread::sleep_for(1ms)` between the two locks in `transferBuggy` to make the deadlock near-certain). Confirm the program freezes, then attach `gdb`/`lldb` and run `thread apply all bt` to see the two threads parked on opposite mutexes.

2. **Fix 3: try-and-back-off.** Add `transferTry` that uses `std::try_lock(a, b)`; if it fails, release everything, sleep a jittered backoff, and retry. This breaks the *no-preemption* condition. Discuss livelock risk and how jittered backoff mitigates it.

3. **Lock hierarchy.** Replace `std::mutex` with the `HierarchicalMutex` from §26.5 (rank accounts by index). Make a thread lock a high index then a low index (violating descent) and confirm it throws a `logic_error` *at the call site* instead of hanging.

4. **Map each fix to a Coffman condition.** For each of the four fixes (ordering, `scoped_lock`, hierarchy, try-back-off), state precisely which of the four conditions it breaks. Note that two of them break the same condition by different means.

5. **Self-transfer trap.** Remove the `if (from == to) return;` guard from `transferOrdered` and pass `from == to`. Explain why locking the same `std::mutex` twice is undefined behavior (self-deadlock), and why `std::scoped_lock` with the same mutex twice is also UB.

6. **TSan in CI.** Wrap the test in a script that builds two binaries — one normal, one `-fsanitize=thread` — and fails the build if TSan reports anything. Add a deliberately racy variant (drop a lock) and confirm CI catches it.

<br><br>

---
---

# PART 5: DEEP DIVE — COMMON QUESTIONS

---
---

<br>

<h2 style="color: #2980B9;">📘 26.9 Q&A</h2>

<br>

#### Q1: "What are the four Coffman conditions, and do I need to break all four?"

Mutual exclusion, hold-and-wait, no-preemption, and circular wait. A deadlock requires **all four** simultaneously, so breaking **any single one** prevents it. In practice you break circular wait (consistent lock ordering) and/or hold-and-wait (`std::scoped_lock`, which acquires all locks atomically). The other two are usually inherent to the design.

<br>

#### Q2: "What's the simplest way to avoid a multi-lock deadlock in C++?"

`std::scoped_lock` (C++17), or `std::lock` + adopting guards (C++11). They acquire several mutexes atomically using a deadlock-free back-off algorithm, regardless of the argument order. It breaks the hold-and-wait condition — the locks are taken together, never one-while-blocking-on-another.

<br>

#### Q3: "When can't I use `std::scoped_lock` and must impose an ordering instead?"

When you can't name all the locks at one point — e.g., you acquire a lock, do work, then *based on that work* decide to acquire another, and you can't hold the first while calling out. Then a documented and (ideally) enforced global ordering is the tool. A lock hierarchy turns the convention into a runtime-checked guarantee.

<br>

#### Q4: "How does a lock hierarchy prevent deadlocks, and why is it better than just a convention?"

It assigns each mutex a level and forbids acquiring a lock unless its level is strictly lower than the lowest you currently hold — enforcing a total order. A convention relies on every developer remembering it; the hierarchy *checks* it at runtime and throws at the exact offending line, converting a rare timing-dependent hang into a deterministic, immediately-debuggable error.

<br>

#### Q5: "How does ThreadSanitizer detect a deadlock that didn't actually happen on this run?"

TSan tracks the *order* in which each thread acquires locks. If it ever sees thread A lock `m1` then `m2`, and thread B lock `m2` then `m1`, it reports a lock-order inversion — a *potential* deadlock — even if the unlucky interleaving that actually freezes never occurred. It's flagging the dangerous structure, not waiting for the bad timing.

<br>

#### Q6: "Why doesn't a deadlocked program crash? How do I even find it?"

Because the threads are blocked, not faulting — they're sitting in `pthread_mutex_lock` waiting politely forever, so the OS sees healthy (if idle) threads. Find it by attaching a debugger to the hung process and running `thread apply all bt`: each deadlocked thread shows a backtrace parked in a lock call, and the cycle of who-waits-for-whom is visible. In testing, TSan or `try_lock_for` timeouts surface it earlier.

<br>

#### Q7: "What is livelock, and how does it differ from deadlock?"

In a **deadlock**, threads are blocked and make no progress. In a **livelock**, threads are *active* — repeatedly retrying — but still make no progress because they keep reacting to each other (e.g., both grab lock A, both fail to get B, both release A and retry in lockstep, forever). The try-and-back-off deadlock fix can cause livelock; the cure is **randomized/jittered backoff** so the threads desynchronize.

<br>

#### Q8: "Can `std::recursive_mutex` solve a deadlock?"

Only the narrow self-deadlock where *one* thread re-locks a mutex it already holds (a locked function calling another locked function on the same object). It does **nothing** for the multi-thread AB-BA deadlock — that's a lock-ordering problem between different threads. Needing a recursive mutex is usually a design smell; prefer restructuring so the lock is taken once.

<br><br>

---

## Reflection Questions

1. List the four Coffman conditions. Why does breaking just one prevent deadlock?
2. Walk through the AB-BA deadlock. Which condition does each of the three fixes break?
3. Why is `std::scoped_lock` deadlock-free, and when can't you use it?
4. How does a lock hierarchy enforce ordering at runtime, and why is that better than a convention?
5. How can ThreadSanitizer warn about a deadlock that didn't actually occur on that run?
6. What is livelock, how does try-and-back-off cause it, and how do you fix it?

---

## Interview Questions

1. "What is a deadlock? State the four conditions required for one."
2. "Write code that deadlocks, then fix it three different ways."
3. "How does `std::scoped_lock`/`std::lock` avoid deadlock when locking multiple mutexes?"
4. "Design a money-transfer function for accounts with per-account mutexes. Avoid deadlock."
5. "What is a lock hierarchy and how do you implement one?"
6. "How does ThreadSanitizer find data races and lock-order inversions? What are its costs?"
7. "A production service hangs intermittently. How do you confirm it's a deadlock and locate it?"
8. "What's the difference between deadlock and livelock? How does try-lock-with-backoff risk livelock?"
9. "Map each deadlock-avoidance strategy to the Coffman condition it breaks."
10. "Can a recursive mutex fix a deadlock? Which kind, and why is needing one a smell?"

---

**Next**: Day 27 — Futures & Promises →
