# Day 25: Reader-Writer Lock

[← Back to Study Plan](../lld-study-plan.md) | [← Day 24](day24-semaphore-barrier.md)

> **Time**: ~1.5-2 hours
> **Goal**: A plain mutex serializes *everything* — but many workloads are read-mostly, and concurrent reads don't conflict. A **reader-writer lock** (shared/exclusive lock) lets *any number of readers* proceed in parallel while guaranteeing a *writer* gets exclusive access. Learn shared vs exclusive modes, the standard `std::shared_mutex` / `std::shared_lock`, the painful trade-off between **read-preferring** and **write-preferring** policies, and why writer starvation is the central design tension. Build a `ReadWriteLock` from scratch with a mutex and condition variables, then stress it with concurrent readers and writers.

---
---

# PART 1: WHY READER-WRITER LOCKS EXIST

---
---

<br>

<h2 style="color: #2980B9;">📘 25.1 The Read-Mostly Workload</h2>

Consider a configuration cache, a routing table, or a DNS map: it is read thousands of times per second and written occasionally. A plain `std::mutex` forces every reader to wait for every other reader — even though two threads *reading* the same data never conflict.

```
With a plain mutex (everything serialized):
  R1 ──┐
  R2 ──┤  must run one at a time, even though they don't conflict
  R3 ──┘

With a reader-writer lock:
  R1 ─┐
  R2 ─┤  all read CONCURRENTLY
  R3 ─┘
  W1     gets EXCLUSIVE access — no reader or other writer runs alongside
```

The insight: a data race needs at least one *writer*. Two concurrent reads are perfectly safe. So we can relax the lock to allow **many readers OR one writer**, never both.

<br>

#### The two modes

| Mode | Also called | Rule |
|------|-------------|------|
| **Shared** (read) | shared lock | Multiple threads may hold it simultaneously |
| **Exclusive** (write) | unique lock | Only one thread may hold it; no shared holders allowed at the same time |

The invariant: at any instant the lock is held by *N readers* (N ≥ 0) **or** *one writer*, but never both at once.

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 25.2 When It Helps — and When It Doesn't</h2>

A reader-writer lock is **not** automatically faster than a plain mutex. It carries more bookkeeping (reader counts, two wait conditions), so it only pays off when:

- **Reads vastly outnumber writes** (e.g., 95%+ reads), and
- **Read critical sections are long enough** that the parallelism gained outweighs the extra bookkeeping.

If reads are tiny (a single field load) or writes are frequent, a plain mutex — or `std::atomic`, or a lock-free snapshot/RCU scheme — often wins. Measure before reaching for it.

| Scenario | Best choice |
|----------|-------------|
| Read-mostly, non-trivial read sections | reader-writer lock |
| Roughly balanced reads/writes | plain mutex (rwlock overhead not worth it) |
| Single scalar value | `std::atomic` |
| Extreme read scaling, rare writes | RCU / copy-on-write snapshots |

<br><br>

---
---

# PART 2: THE STANDARD FACILITY

---
---

<br>

<h2 style="color: #2980B9;">📘 25.3 std::shared_mutex</h2>

C++17 provides `std::shared_mutex` (and C++14's `std::shared_timed_mutex`). It exposes two locking modes:

```cpp
#include <shared_mutex>

std::shared_mutex rw;

// Exclusive (writer):
rw.lock();      /* ... write ... */   rw.unlock();

// Shared (reader):
rw.lock_shared();  /* ... read ... */  rw.unlock_shared();
```

As always, lock *manually* only at your peril. Use the RAII wrappers:

```cpp
// Writer — exclusive
{
    std::unique_lock<std::shared_mutex> lk(rw);    // or std::lock_guard
    data.modify();
}

// Reader — shared
{
    std::shared_lock<std::shared_mutex> lk(rw);    // shared_lock = the "read lock" RAII
    use(data.read());
}
```

`std::shared_lock` is to shared mode what `std::unique_lock` is to exclusive mode: an RAII guard that acquires shared on construction and releases on destruction.

<br>

#### A thread-safe map with shared_mutex

```cpp
template <typename K, typename V>
class ConcurrentMap {
    mutable std::shared_mutex rw_;
    std::unordered_map<K, V> map_;
public:
    std::optional<V> get(const K& k) const {
        std::shared_lock lk(rw_);          // many readers in parallel
        if (auto it = map_.find(k); it != map_.end()) return it->second;
        return std::nullopt;
    }
    void put(const K& k, V v) {
        std::unique_lock lk(rw_);          // exclusive — blocks all readers and writers
        map_[k] = std::move(v);
    }
};
```

`get` takes a shared lock (reads scale); `put` takes an exclusive lock (writes serialize). `mutable` again lets the `const` getter lock.

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 25.4 The Upgrade Problem</h2>

A tempting pattern is "take a read lock, decide I need to write, *upgrade* to a write lock without releasing." **C++'s `std::shared_mutex` does not support atomic upgrade** — and for good reason: if two readers both try to upgrade at once, each waits for the other to drop its shared lock, and they deadlock.

The safe pattern is **release-then-reacquire, and re-check**:

```cpp
{
    std::shared_lock rlk(rw_);
    if (cache.contains(k)) return cache.get(k);   // fast read path
}                                                  // drop the read lock
{
    std::unique_lock wlk(rw_);
    if (!cache.contains(k))                        // RE-CHECK — another writer may have inserted
        cache.insert(k, compute(k));
    return cache.get(k);
}
```

The re-check matters because between dropping the read lock and taking the write lock, another thread may have done the work. (Boost offers an `upgrade_lock`/`upgrade_to_unique_lock` mechanism that allows at most one upgrader at a time; the standard deliberately omits it.)

<br><br>

---
---

# PART 3: POLICY — THE STARVATION TRADE-OFF

---
---

<br>

<h2 style="color: #2980B9;">📘 25.5 Read-Preferring vs Write-Preferring</h2>

When both readers and writers are waiting, *who goes next?* This policy choice is the soul of a reader-writer lock.

```
Read-preferring (readers win):
  A writer is waiting, but new readers keep arriving and are let in.
  → maximum read throughput, but the writer may NEVER run → WRITER STARVATION

Write-preferring (writers win):
  Once a writer is waiting, new readers BLOCK (even though existing readers finish).
  → writers make progress, but a burst of writes can stall readers → reduced read parallelism
```

| Policy | Behavior | Pro | Con |
|--------|----------|-----|-----|
| **Read-preferring** | New readers may join even while a writer waits | Highest read throughput | **Writer starvation** under continuous reads |
| **Write-preferring** | A waiting writer blocks newly arriving readers | Writers always make progress | Lower read parallelism; possible reader stall under write bursts |
| **Fair / FIFO** | Serve in arrival order | No starvation either way | Most bookkeeping; lowest peak throughput |

<br>

#### Why writer starvation is the famous failure

A naive reader-writer lock that simply "let readers in whenever count permits" is read-preferring by accident. In a read-heavy system, there is *always* at least one reader holding the lock, so a writer waiting for the reader count to hit zero may wait **forever**. The classic fix: when a writer is waiting, *stop admitting new readers* — drain the current readers, let the writer in, then resume readers. That's write-preferring, and it's the usual sane default for general-purpose locks.

<br>

#### What does std::shared_mutex do?

The standard **does not specify** the policy — it is implementation-defined, and implementations vary (some lean read-preferring, some write-preferring, some fair). If your correctness depends on starvation-freedom, do not rely on the standard mutex's unspecified behavior; build a lock with the policy you need (today's exercise) or restructure the workload.

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 25.6 Other Hazards</h2>

- **Recursive shared locking can deadlock.** If a thread holds a shared lock and a writer is now waiting (write-preferring), and the same thread tries to take the shared lock *again*, it blocks behind the waiting writer — which is waiting for *it* to release. Self-deadlock. Don't re-enter a shared lock.
- **Holding a read lock while calling out** to code that might take the write lock (directly or via a callback) is a lock-ordering deadlock. The same "do little under the lock" discipline from earlier days applies, doubled.
- **Don't write under a shared lock.** A shared lock grants *read* permission only. Mutating data while holding it shared (because "I'm pretty sure no one else is writing") is a data race. Take the exclusive lock to write.

<br><br>

---
---

# PART 4: BUILD EXERCISE

---
---

<br>

<h2 style="color: #2980B9;">📘 25.7 Exercise: Write-Preferring ReadWriteLock from Scratch</h2>

Build a `ReadWriteLock` using a mutex and condition variables, with a **write-preferring** policy (no writer starvation). Provide RAII guards. Wrap a shared value, then stress it with many readers and a few writers, verifying that readers never observe a torn/partial write and that writers always make progress.

<br>

#### Skeleton (header)

```cpp
// rwlock.h
#pragma once
#include <mutex>
#include <condition_variable>
#include <cstddef>

// Write-preferring reader-writer lock.
//   Invariant: activeWriters_ is 0 or 1; when 1, activeReaders_ must be 0.
//   Write-preferring: a reader will NOT enter while any writer is waiting
//   or active, so a continuous stream of readers cannot starve a writer.
class ReadWriteLock {
    std::mutex m_;
    std::condition_variable readerCv_;   // readers wait here
    std::condition_variable writerCv_;   // writers wait here

    std::size_t activeReaders_ = 0;
    std::size_t waitingWriters_ = 0;
    bool        activeWriter_  = false;

public:
    void lockShared() {
        std::unique_lock<std::mutex> lk(m_);
        // Block if a writer is active OR waiting (write preference).
        readerCv_.wait(lk, [&]{ return !activeWriter_ && waitingWriters_ == 0; });
        ++activeReaders_;
    }

    void unlockShared() {
        std::unique_lock<std::mutex> lk(m_);
        if (--activeReaders_ == 0 && waitingWriters_ > 0) {
            lk.unlock();
            writerCv_.notify_one();   // last reader out → let a waiting writer in
        }
    }

    void lock() {                     // exclusive
        std::unique_lock<std::mutex> lk(m_);
        ++waitingWriters_;
        writerCv_.wait(lk, [&]{ return !activeWriter_ && activeReaders_ == 0; });
        --waitingWriters_;
        activeWriter_ = true;
    }

    void unlock() {                   // exclusive
        std::unique_lock<std::mutex> lk(m_);
        activeWriter_ = false;
        // Prefer waiting writers; otherwise release the herd of readers.
        if (waitingWriters_ > 0) {
            lk.unlock();
            writerCv_.notify_one();
        } else {
            lk.unlock();
            readerCv_.notify_all();
        }
    }
};

// ── RAII guards ─────────────────────────────────────────
class ReadGuard {
    ReadWriteLock& lk_;
public:
    explicit ReadGuard(ReadWriteLock& lk) : lk_(lk) { lk_.lockShared(); }
    ~ReadGuard() { lk_.unlockShared(); }
    ReadGuard(const ReadGuard&) = delete;
    ReadGuard& operator=(const ReadGuard&) = delete;
};

class WriteGuard {
    ReadWriteLock& lk_;
public:
    explicit WriteGuard(ReadWriteLock& lk) : lk_(lk) { lk_.lock(); }
    ~WriteGuard() { lk_.unlock(); }
    WriteGuard(const WriteGuard&) = delete;
    WriteGuard& operator=(const WriteGuard&) = delete;
};
```

<br>

#### Test driver

```cpp
// main.cpp
#include "rwlock.h"
#include <thread>
#include <vector>
#include <atomic>
#include <array>
#include <iostream>
#include <cassert>

int main() {
    std::cout << "=== Write-preferring reader-writer lock ===\n";

    ReadWriteLock rw;

    // Shared data: a 3-element array that writers keep internally consistent
    // (all three elements equal). A reader must never see a torn write
    // (elements disagreeing) — that would prove a reader ran during a write.
    std::array<long long, 3> data = {0, 0, 0};

    constexpr int kReaders = 12;
    constexpr int kWriters = 3;
    constexpr int kReadOps = 200000;
    constexpr int kWriteOps = 5000;

    std::atomic<long long> tornReads{0};
    std::atomic<long long> readCount{0};
    std::atomic<long long> writeCount{0};

    std::vector<std::thread> threads;

    // Writers: bump all three elements together under the exclusive lock.
    for (int w = 0; w < kWriters; ++w) {
        threads.emplace_back([&]{
            for (int i = 0; i < kWriteOps; ++i) {
                WriteGuard g(rw);
                long long next = data[0] + 1;
                data[0] = next;            // a reader interleaving here would see a torn state
                data[1] = next;
                data[2] = next;
                ++writeCount;
            }
        });
    }

    // Readers: verify all three elements agree (atomic snapshot under shared lock).
    for (int r = 0; r < kReaders; ++r) {
        threads.emplace_back([&]{
            for (int i = 0; i < kReadOps; ++i) {
                ReadGuard g(rw);
                if (data[0] != data[1] || data[1] != data[2])
                    ++tornReads;
                ++readCount;
            }
        });
    }

    for (auto& t : threads) t.join();

    std::cout << "  reads = " << readCount.load()
              << ", writes = " << writeCount.load() << "\n";
    std::cout << "  torn reads observed = " << tornReads.load() << "\n";
    std::cout << "  final value = " << data[0] << "\n";

    assert(tornReads.load() == 0);                               // no reader saw a write in progress
    assert(writeCount.load() == kWriters * kWriteOps);           // every writer made progress (no starvation)
    assert(data[0] == data[1] && data[1] == data[2]);            // consistent final state
    assert(data[0] == static_cast<long long>(kWriters) * kWriteOps);

    std::cout << "All assertions passed.\n";
    return 0;
}
```

<br>

#### Compile and run

```bash
g++ -std=c++17 -Wall -Wextra -Wpedantic -O2 -pthread \
    -o day25 main.cpp && ./day25
```

#### Under ThreadSanitizer

```bash
g++ -std=c++17 -Wall -Wextra -Wpedantic -g -fsanitize=thread -pthread \
    -o day25_tsan main.cpp && ./day25_tsan
```

The `tornReads == 0` assertion is the correctness oracle: if a reader ever ran concurrently with a writer, it would see `data[0..2]` disagree mid-update. The `writeCount` assertion proves writers were never starved (all completed). TSan independently confirms there's no data race.

<br>

#### Expected output

```
=== Write-preferring reader-writer lock ===
  reads = 2400000, writes = 15000
  torn reads observed = 0
  final value = 15000
All assertions passed.
```

<br>

#### Bonus Challenges

1. **Build the read-preferring version.** Remove the `waitingWriters_ == 0` term from the reader's wait predicate so new readers always get in. Then run a test with continuous readers and one writer; instrument how long the writer waits. Demonstrate (with a timeout) that the writer can starve.

2. **Compare against `std::shared_mutex`.** Swap your `ReadWriteLock` for `std::shared_mutex` + `std::shared_lock`/`unique_lock`. Benchmark read-heavy and write-heavy workloads. Note that the standard's policy is unspecified — does your platform's seem read- or write-preferring?

3. **Compare against a plain mutex.** Replace the rwlock with a single `std::mutex` for all access. At what read/write ratio does the rwlock start beating the plain mutex? (Hint: with tiny critical sections, the mutex often wins.)

4. **Fair (FIFO) policy.** Implement a ticketing scheme so readers and writers are served in arrival order. Show it eliminates both writer *and* reader starvation, then measure the throughput cost versus write-preferring.

5. **The upgrade trap.** Try to add an `upgrade()` that turns a held shared lock into an exclusive one without releasing. Construct the two-readers-both-upgrading deadlock. Then implement the correct release-then-reacquire-and-recheck pattern instead.

6. **Reader count metrics.** Track and print the maximum number of concurrent readers actually observed during the run (use an atomic max). Confirm it exceeds 1 (proving real read parallelism) and that it drops to 0 whenever a writer holds the lock.

<br><br>

---
---

# PART 5: DEEP DIVE — COMMON QUESTIONS

---
---

<br>

<h2 style="color: #2980B9;">📘 25.8 Q&A</h2>

<br>

#### Q1: "When is a reader-writer lock actually worth it over a plain mutex?"

Only when reads vastly dominate writes *and* read critical sections are long enough that running them in parallel saves more than the lock's extra bookkeeping costs. For tiny reads (a single field) or balanced read/write ratios, a plain mutex is usually faster, and a single scalar is better served by `std::atomic`. Always measure; the rwlock is not a free upgrade.

<br>

#### Q2: "What's the invariant a reader-writer lock maintains?"

At any instant: either *N* readers (N ≥ 0) hold it in shared mode, **or** exactly one writer holds it in exclusive mode — never both. Concurrent readers are safe because reads don't conflict; a writer needs exclusivity because a write concurrent with any other access is a data race.

<br>

#### Q3: "What is writer starvation and how do I prevent it?"

In a read-preferring lock, new readers are admitted whenever the reader count allows — so in a read-heavy system the reader count never reaches zero and a waiting writer waits forever. Prevent it with a **write-preferring** policy: once a writer is waiting, block newly arriving readers, drain the current readers, admit the writer, then resume readers. A fair/FIFO policy also works but costs more.

<br>

#### Q4: "What policy does `std::shared_mutex` use?"

Unspecified by the standard — it's implementation-defined and varies across platforms and library versions. If your program's correctness depends on starvation-freedom, you cannot rely on the standard mutex; build a lock with the explicit policy you need, or restructure to avoid the dependency.

<br>

#### Q5: "Why can't I atomically upgrade a shared lock to an exclusive lock?"

Because two threads both holding the shared lock could both request the upgrade simultaneously: each must wait for the other to drop its shared lock before it can become exclusive — a deadlock. The standard `std::shared_mutex` therefore offers no upgrade. The safe pattern is release the shared lock, take the exclusive lock, and *re-check* the condition (another thread may have made the change in the gap).

<br>

#### Q6: "Can I modify data while holding only a shared lock?"

No. A shared lock grants read permission only; multiple readers hold it concurrently, so writing under it is a data race with those other readers. To write, you must take the exclusive lock — even if you "think" no one else is writing right now.

<br>

#### Q7: "Is recursive shared locking safe?"

It's dangerous. With a write-preferring lock, if a thread holds a shared lock, a writer starts waiting, and the same thread tries to take the shared lock again, the second acquisition blocks behind the waiting writer — which is itself waiting for that thread to release. Self-deadlock. Treat reader-writer locks as non-recursive and never re-enter a shared lock.

<br>

#### Q8: "What are the alternatives when even an rwlock isn't fast enough for extreme read scaling?"

**Copy-on-write / snapshots**: writers build a new immutable version and atomically swap a pointer; readers grab the current pointer lock-free and read a consistent snapshot. **RCU (read-copy-update)**: readers run with essentially zero synchronization; writers publish new versions and defer reclamation until all readers of the old version have finished. Both trade memory and write complexity for near-perfect read scalability, and are how kernels and high-read systems handle hot, read-mostly data.

<br><br>

---

## Reflection Questions

1. Why can multiple readers safely share a lock while a writer needs exclusivity? What property of a data race makes this sound?
2. When does a reader-writer lock beat a plain mutex, and when does it lose?
3. Explain read-preferring vs write-preferring. Which causes writer starvation, and how does the other fix it?
4. Why is atomic lock upgrade unsafe, and what is the correct release-reacquire-recheck pattern?
5. Why is `std::shared_mutex`'s policy being unspecified a problem for starvation-sensitive code?
6. When would you abandon an rwlock for copy-on-write snapshots or RCU?

---

## Interview Questions

1. "Implement a reader-writer lock from a mutex and condition variables."
2. "What is writer starvation? Make your lock write-preferring and explain the change."
3. "When is a reader-writer lock worse than a plain mutex?"
4. "What does `std::shared_mutex` guarantee about read/write preference?"
5. "Why can't you atomically upgrade a shared lock to an exclusive one?"
6. "What's the invariant a reader-writer lock maintains? How do you enforce it?"
7. "Compare read-preferring, write-preferring, and fair policies."
8. "Can you write while holding a shared lock? Why or why not?"
9. "How does copy-on-write or RCU scale reads better than a reader-writer lock?"
10. "Design a concurrent configuration cache read thousands of times per second and updated rarely. What synchronization do you choose and why?"

---

**Next**: Day 26 — Deadlocks & Debugging →
