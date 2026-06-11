# Day 39: LRU Cache — Thread-Safe

[← Back to Study Plan](../lld-study-plan.md) | [← Day 38](day38-lru-cache.md)

> **Time**: ~1.5-2 hours
> **Goal**: Yesterday's LRU cache is correct single-threaded but **explodes under concurrency** — `get` mutates the recency list, so two threads reading at once race on the same pointers. Today you make it thread-safe and confront the central irony: **even reads write** in an LRU cache, so a naive `shared_mutex` read lock is *wrong*. You'll learn coarse vs fine-grained locking, why the obvious `shared_mutex` optimization fails, **sharding** as the scalable answer, and approximate-LRU tricks that recover read concurrency. Then you'll build a sharded thread-safe cache and benchmark it against a single-mutex version under concurrent load.

---
---

# PART 1: WHY THE NAIVE CACHE BREAKS

---
---

<br>

<h2 style="color: #2980B9;">📘 39.1 The Data Race</h2>

Recall yesterday's `get`:

```cpp
std::optional<V> get(const K& key) {
    auto it = m_map.find(key);
    if (it == m_map.end()) return std::nullopt;
    m_list.splice(m_list.begin(), m_list, it->second);  // ← MUTATES the list
    return it->second->second;
}
```

`splice` rewrites the `prev`/`next` pointers of several nodes. If thread A splices node X to the front while thread B simultaneously splices node Y (or even reads the head), they trample each other's pointer writes:

```
  Thread A: head->next = X;  X->prev = head;   ...
  Thread B: head->next = Y;  Y->prev = head;   ...   ← interleaves → corrupt list

  Result: head->next points to one node, but that node's prev points elsewhere.
          Iteration loops forever, or dereferences a freed node → crash / UB.
```

And `unordered_map` itself is not safe for concurrent modification. **Both** structures are being mutated on every `get`. So the cache needs synchronization on *all* operations — including reads. That single fact drives the entire day.

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 39.2 The `shared_mutex` Trap</h2>

The instinctive optimization for a read-heavy structure: a `std::shared_mutex` (reader-writer lock). Many readers share; writers get exclusive access.

```cpp
std::shared_mutex m_mtx;

std::optional<V> get(const K& key) {
    std::shared_lock lk(m_mtx);          // ← SHARED (read) lock ... WRONG for LRU
    auto it = m_map.find(key);
    if (it == m_map.end()) return std::nullopt;
    m_list.splice(...);                  // ← but this is a WRITE under a read lock!
    return it->second->second;
}
```

This compiles and *appears* to work, then corrupts memory under load. The reasoning failure: a `shared_lock` permits **multiple threads in `get` at once**, but each one is *writing* to the shared list via `splice`. A read lock provides no mutual exclusion against other readers — so two `get`s splice concurrently and the list is destroyed.

> **The lesson:** "read-heavy ⇒ use `shared_mutex`" is only valid if your reads are *actually reads*. In an LRU cache, **`get` is a write** (it reorders recency). A `shared_mutex` is the right tool only if you can make `get` genuinely non-mutating — which requires giving up exact recency (see §39.6).

<br>

So the *correct simple* answer is a plain `std::mutex` held exclusively on every operation, reads included.

<br><br>

---
---

# PART 2: COARSE-GRAINED LOCKING

---
---

<br>

<h2 style="color: #2980B9;">📘 39.3 One Mutex to Rule It All</h2>

The simplest correct design: wrap the entire single-threaded cache and take one exclusive lock per operation.

```cpp
#include <mutex>

template <typename K, typename V>
class LockedLRUCache {
    LRUCache<K, V>     m_cache;   // yesterday's single-threaded cache
    mutable std::mutex m_mtx;

public:
    explicit LockedLRUCache(std::size_t cap) : m_cache(cap) {}

    std::optional<V> get(const K& key) {
        std::lock_guard<std::mutex> lk(m_mtx);
        return m_cache.get(key);
    }
    void put(const K& key, V value) {
        std::lock_guard<std::mutex> lk(m_mtx);
        m_cache.put(key, std::move(value));
    }
};
```

| Pros | Cons |
|------|------|
| Trivially correct — composes the proven single-threaded cache | **Serializes everything** — one operation at a time, no matter how many cores |
| Tiny code; nothing to get wrong | Becomes the bottleneck under contention; cores spin waiting on the lock |
| No risk of deadlock (one lock) | Throughput is capped at ~1 / (lock acquire + op + release) |

This is the right *first* implementation: ship it, measure it, and only add complexity if profiling shows the lock is hot. For many caches it's entirely sufficient.

<br>

#### Don't return references under the lock

```cpp
const V& get(const K& key) {                       // DANGEROUS
    std::lock_guard<std::mutex> lk(m_mtx);
    return m_cache.getRef(key);                     // lock released on return!
}
```

The lock is released when `lk` is destroyed at the `return`, but the caller now holds a reference into the cache that another thread can evict/mutate → dangling reference. Thread-safe getters must return **by value** (a copy), not by reference. `std::optional<V>` is the right return type.

<br><br>

---
---

# PART 3: SHARDING — THE SCALABLE ANSWER

---
---

<br>

<h2 style="color: #2980B9;">📘 39.4 Partition the Keyspace</h2>

Since every operation needs an exclusive lock, the way to get parallelism is to have **many independent locks**, each guarding a slice of the keys. **Shard** the cache into N sub-caches; a key maps to exactly one shard via its hash. Operations on different shards proceed in parallel.

```
                      hash(key) % N
   put("alpha") ──────────┐
                          ▼
   ┌──────────┬──────────┬──────────┬──────────┐
   │ Shard 0  │ Shard 1  │ Shard 2  │ Shard 3  │   N independent shards
   │ mutex 0  │ mutex 1  │ mutex 2  │ mutex 3  │   N independent locks
   │ LRU 0    │ LRU 1    │ LRU 2    │ LRU 3    │   each a full LRU cache
   └──────────┴──────────┴──────────┴──────────┘
        ▲          ▲
   thread A    thread B  ← different shards → NO contention, run in parallel
```

A given key always lands in the same shard, so each shard is a self-contained LRU with its own mutex and its own capacity (`totalCap / N`). Contention drops by roughly a factor of N (assuming keys spread evenly across shards).

```cpp
template <typename K, typename V, typename Hash = std::hash<K>>
class ShardedLRUCache {
    struct Shard {
        std::mutex     mtx;
        LRUCache<K, V> cache;
        explicit Shard(std::size_t cap) : cache(cap) {}
    };
    std::vector<std::unique_ptr<Shard>> m_shards;
    Hash                                m_hash;

    Shard& shardFor(const K& key) {
        std::size_t idx = m_hash(key) & (m_shards.size() - 1);  // size is power of 2
        return *m_shards[idx];
    }

public:
    ShardedLRUCache(std::size_t totalCapacity, std::size_t numShards = 16) {
        // round numShards up to a power of two so we can mask instead of modulo
        std::size_t n = 1;
        while (n < numShards) n <<= 1;
        std::size_t perShard = std::max<std::size_t>(1, totalCapacity / n);
        m_shards.reserve(n);
        for (std::size_t i = 0; i < n; ++i)
            m_shards.push_back(std::make_unique<Shard>(perShard));
    }

    std::optional<V> get(const K& key) {
        Shard& s = shardFor(key);
        std::lock_guard<std::mutex> lk(s.mtx);
        return s.cache.get(key);
    }
    void put(const K& key, V value) {
        Shard& s = shardFor(key);
        std::lock_guard<std::mutex> lk(s.mtx);
        s.cache.put(key, std::move(value));
    }
};
```

<br>

#### Sharding trade-offs

| Pros | Cons |
|------|------|
| Near-linear scaling with shard count (under even key distribution) | Eviction is **per-shard**, not global — a hot shard evicts while others have room |
| Each shard is just the proven single-threaded cache + a mutex | Slightly more memory (N lists, N maps, N locks) |
| No cross-shard coordination → no deadlock | Hot keys mapping to one shard reintroduce contention on that shard |
| Power-of-two shard count → `&` mask instead of `%` modulo | LRU is now *approximate globally* — the global LRU entry might not be the one evicted |

The "per-shard eviction" caveat is real but usually acceptable: with a good hash and enough shards, load spreads evenly and the approximation barely affects hit ratio. This is exactly how Java's `ConcurrentHashMap` (segmented locking), `Caffeine`, and many production caches scale.

<br>

> **Why power-of-two shards?** `hash & (N-1)` is a single AND instruction; `hash % N` is a (much slower) division. The trick only works when `N` is a power of two, hence the rounding in the constructor.

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 39.5 Picking the Shard Count</h2>

- Too few shards → contention persists.
- Too many shards → wasted memory, and each shard's tiny capacity hurts hit ratio (an entry can be globally hot but evicted from its small shard).

A common heuristic: `numShards ≈ 4 × hardware_concurrency`, rounded to a power of two, capped so each shard still holds a reasonable number of entries (say ≥ 64). Measure: plot throughput vs shard count and find the knee.

<br><br>

---
---

# PART 4: RECOVERING READ CONCURRENCY

---
---

<br>

<h2 style="color: #2980B9;">📘 39.6 Approximate LRU — Make `get` a Real Read</h2>

If you truly want a `shared_mutex` (many concurrent readers), you must make `get` **not mutate** the shared structure. The standard trick: **defer recency updates**.

Approaches used in real systems:

| Technique | How `get` avoids writing | Cost |
|-----------|--------------------------|------|
| **Clock / second-chance** | `get` just sets an atomic "referenced" bit on the entry (no list move). Eviction scans and gives bit-set entries a second chance. | Approximate LRU; O(1) amortized eviction |
| **Per-entry access timestamp** | `get` stores `now()` in an atomic field; eviction picks the oldest timestamp (sampled, not exact) | Approximate; eviction must sample/scan |
| **Batched recency log** | `get` appends `(key)` to a thread-local ring; a background thread later applies the moves to the list | Exact-ish, but delayed; extra thread |
| **Sampled eviction (Redis-style)** | No ordering at all; on eviction, sample K random keys, evict the oldest among them | Approximate; no per-read write |

Caffeine (the high-performance Java cache) uses ring buffers to record reads, draining them to update an LRU/LFU structure asynchronously — so `get` is lock-free and writes only to a thread-local buffer. **Clock** is the OS page-cache classic: a referenced bit instead of a list move.

```
   Clock (second-chance) eviction:
   entries arranged in a ring; a "hand" sweeps.
   get(k):  set k.referenced = 1        (atomic store — no list surgery)
   evict():
       while (hand.referenced == 1) { hand.referenced = 0; advance(); }  // second chance
       evict(hand); advance();
```

The trade-off everywhere: **give up exact LRU ordering to gain concurrent reads.** For most caches the hit-ratio difference between exact and approximate LRU is negligible, so this is a clear win when reads dominate.

<br>

> **The takeaway:** you can have a `shared_mutex` (or even lock-free reads), but *only* after redesigning `get` to not mutate shared state. The exact-LRU + `std::list::splice` design fundamentally cannot use a read lock. Sharding (Part 3) keeps exact-per-shard LRU and scales without that redesign — which is why it's the most common production answer.

<br><br>

---
---

# PART 5: BUILD EXERCISE

---
---

<br>

<h2 style="color: #2980B9;">📘 39.7 Exercise: Sharded Thread-Safe LRU + Benchmark</h2>

Wrap yesterday's `LRUCache` in (a) a single-mutex version and (b) a sharded version. Hammer both with many threads doing concurrent `get`/`put`, verify correctness under ThreadSanitizer, and measure throughput to see sharding's win.

<br>

#### Skeleton

```cpp
// ts_lru.h
#pragma once
#include <algorithm>
#include <cstddef>
#include <list>
#include <memory>
#include <mutex>
#include <optional>
#include <unordered_map>
#include <utility>
#include <vector>

// ── Single-threaded core (Day 38) ───────────────────────
template <typename K, typename V>
class LRUCache {
    using Entry  = std::pair<K, V>;
    using ListIt = typename std::list<Entry>::iterator;
    std::size_t                   m_cap;
    std::list<Entry>              m_list;
    std::unordered_map<K, ListIt> m_map;
public:
    explicit LRUCache(std::size_t cap) : m_cap(cap) {}
    std::optional<V> get(const K& key) {
        auto it = m_map.find(key);
        if (it == m_map.end()) return std::nullopt;
        m_list.splice(m_list.begin(), m_list, it->second);
        return it->second->second;
    }
    void put(const K& key, V value) {
        auto it = m_map.find(key);
        if (it != m_map.end()) {
            it->second->second = std::move(value);
            m_list.splice(m_list.begin(), m_list, it->second);
            return;
        }
        m_list.emplace_front(key, std::move(value));
        m_map[key] = m_list.begin();
        if (m_map.size() > m_cap) {
            auto& v = m_list.back();
            m_map.erase(v.first);
            m_list.pop_back();
        }
    }
    std::size_t size() const { return m_map.size(); }
};

// ── (a) Coarse single-mutex wrapper ─────────────────────
template <typename K, typename V>
class LockedLRUCache {
    LRUCache<K, V>     m_cache;
    mutable std::mutex m_mtx;
public:
    explicit LockedLRUCache(std::size_t cap) : m_cache(cap) {}
    std::optional<V> get(const K& key) {
        std::lock_guard<std::mutex> lk(m_mtx);
        return m_cache.get(key);
    }
    void put(const K& key, V value) {
        std::lock_guard<std::mutex> lk(m_mtx);
        m_cache.put(key, std::move(value));
    }
};

// ── (b) Sharded wrapper ─────────────────────────────────
template <typename K, typename V, typename Hash = std::hash<K>>
class ShardedLRUCache {
    struct Shard {
        std::mutex     mtx;
        LRUCache<K, V> cache;
        explicit Shard(std::size_t cap) : cache(cap) {}
    };
    std::vector<std::unique_ptr<Shard>> m_shards;
    Hash                                m_hash;
    Shard& shardFor(const K& key) {
        std::size_t idx = m_hash(key) & (m_shards.size() - 1);
        return *m_shards[idx];
    }
public:
    ShardedLRUCache(std::size_t totalCapacity, std::size_t numShards = 16) {
        std::size_t n = 1;
        while (n < numShards) n <<= 1;
        std::size_t perShard = std::max<std::size_t>(1, totalCapacity / n);
        m_shards.reserve(n);
        for (std::size_t i = 0; i < n; ++i)
            m_shards.push_back(std::make_unique<Shard>(perShard));
    }
    std::optional<V> get(const K& key) {
        Shard& s = shardFor(key);
        std::lock_guard<std::mutex> lk(s.mtx);
        return s.cache.get(key);
    }
    void put(const K& key, V value) {
        Shard& s = shardFor(key);
        std::lock_guard<std::mutex> lk(s.mtx);
        s.cache.put(key, std::move(value));
    }
};
```

<br>

#### Test + benchmark driver

```cpp
// main.cpp
#include "ts_lru.h"
#include <atomic>
#include <cassert>
#include <chrono>
#include <iostream>
#include <random>
#include <thread>

template <typename Cache>
double hammer(Cache& cache, int numThreads, int opsPerThread, int keyspace) {
    std::atomic<long> hits{0};
    auto worker = [&](unsigned seed) {
        std::mt19937 rng(seed);
        std::uniform_int_distribution<int> keyDist(0, keyspace - 1);
        std::uniform_int_distribution<int> opDist(0, 99);
        long localHits = 0;
        for (int i = 0; i < opsPerThread; ++i) {
            int k = keyDist(rng);
            if (opDist(rng) < 80) {                     // 80% reads
                if (cache.get(k).has_value()) ++localHits;
            } else {                                    // 20% writes
                cache.put(k, k * 10);
            }
        }
        hits.fetch_add(localHits, std::memory_order_relaxed);
    };

    auto t0 = std::chrono::steady_clock::now();
    std::vector<std::thread> ts;
    for (int t = 0; t < numThreads; ++t) ts.emplace_back(worker, 1234u + t);
    for (auto& t : ts) t.join();
    auto t1 = std::chrono::steady_clock::now();

    double secs = std::chrono::duration<double>(t1 - t0).count();
    long totalOps = static_cast<long>(numThreads) * opsPerThread;
    std::cout << "  hits=" << hits.load() << "  ops=" << totalOps
              << "  time=" << secs << "s"
              << "  throughput=" << (totalOps / secs / 1e6) << " Mops/s\n";
    return secs;
}

int main() {
    const int    threads = 8;
    const int    ops     = 200000;
    const int    keys    = 5000;
    const std::size_t cap = 2000;

    // Correctness: a single-threaded sanity check first
    {
        ShardedLRUCache<int, int> c(cap, 16);
        c.put(42, 420);
        assert(c.get(42).value() == 420);
        assert(c.get(99999) == std::nullopt);
        std::cout << "[correctness] basic ops OK\n";
    }

    std::cout << "\n=== Single mutex (coarse) ===\n";
    {
        LockedLRUCache<int, int> c(cap);
        hammer(c, threads, ops, keys);
    }

    std::cout << "\n=== Sharded (16 shards) ===\n";
    {
        ShardedLRUCache<int, int> c(cap, 16);
        hammer(c, threads, ops, keys);
    }

    std::cout << "\nDone. Sharded should show higher throughput on a multi-core box.\n";
    return 0;
}
```

<br>

#### Compile and run

```bash
# Correctness pass (catch data races):
g++ -std=c++17 -Wall -Wextra -Wpedantic -fsanitize=thread \
    -pthread -O1 -o day39_tsan main.cpp && ./day39_tsan

# Performance pass (real numbers — sanitizers distort timing):
g++ -std=c++17 -Wall -Wextra -Wpedantic -O2 \
    -pthread -o day39 main.cpp && ./day39
```

Run the TSan build for correctness and the `-O2` build (no sanitizer) for the throughput comparison — sanitizers add huge overhead that masks the sharding speedup.

<br>

#### Expected output pattern

```
[correctness] basic ops OK

=== Single mutex (coarse) ===
  hits=...  ops=1600000  time=0.21s  throughput=7.6 Mops/s

=== Sharded (16 shards) ===
  hits=...  ops=1600000  time=0.05s  throughput=30+ Mops/s

Done. Sharded should show higher throughput on a multi-core box.
```

Exact numbers vary by machine, but the sharded version should be several times faster on a multi-core CPU because threads touching different shards never contend. Under TSan, both must report **zero data races**.

<br>

#### Bonus Challenges

1. **`shared_mutex` done wrong, then right** — first show (with TSan) that a `shared_lock` around the splicing `get` *races*. Then implement the **Clock** variant where `get` only sets an atomic referenced bit, and show a `shared_mutex` (or fully lock-free reads) now works.

2. **Shard-count sweep** — benchmark 1, 2, 4, 8, 16, 64, 256 shards and plot throughput. Find the knee and explain why more shards eventually stops helping (and can hurt hit ratio).

3. **Hot-key contention** — make 90% of accesses hit a single key. Show that sharding *doesn't* help (all hot accesses land on one shard) and discuss mitigations (per-thread cache of hot keys).

4. **`try_lock` fast path** — on `get`, attempt `try_lock`; if it fails (shard busy), skip the recency update and read a snapshot, avoiding the wait. Measure the latency-tail improvement vs hit-ratio cost.

5. **Sampled eviction (Redis-style)** — replace per-shard exact LRU with: on eviction, sample K random keys from the shard and evict the oldest. Compare hit ratio and write throughput.

6. **Per-shard stats aggregation** — add hit/miss counters per shard and a method that sums them. Watch for false sharing — pad counters to a cache line and measure the difference.

<br><br>

---
---

# PART 6: DEEP DIVE — COMMON QUESTIONS

---
---

<br>

<h2 style="color: #2980B9;">📘 39.8 Q&A</h2>

<br>

#### Q1: "Why can't I use a `shared_mutex` read lock for `get`?"

Because `get` *writes*: it splices the accessed node to the front of the recency list. A `shared_lock` lets multiple threads into `get` simultaneously, and they all mutate the same list → data race and corruption. A read lock is only valid if the read truly doesn't modify shared state, which exact LRU's `get` violates.

<br>

#### Q2: "So what's the simplest correct thread-safe LRU?"

A single `std::mutex` held exclusively on **every** operation, reads included. It's trivially correct and often fast enough. Its weakness is that it serializes all access — no parallelism — which is what sharding addresses.

<br>

#### Q3: "How does sharding scale better?"

It partitions the keyspace into N independent sub-caches, each with its own lock. A key hashes to one shard, so operations on different shards run fully in parallel. Contention drops ~N-fold under even key distribution. This is segmented locking — the model behind `ConcurrentHashMap` and most production caches.

<br>

#### Q4: "What does sharding give up?"

Global LRU exactness. Eviction is per-shard, so the entry evicted from a full shard may not be the globally least-recently-used entry. With enough shards and a good hash, load spreads evenly and hit ratio barely suffers. It also costs a bit more memory (N lists/maps/locks) and doesn't help when one key is overwhelmingly hot (all its traffic hits one shard).

<br>

#### Q5: "How do I pick the shard count?"

Round to a power of two so you can mask (`hash & (N-1)`) instead of modulo. A heuristic is ~4× hardware concurrency, capped so each shard still holds enough entries (≥ ~64) to maintain hit ratio. Then measure: throughput vs shard count has a knee — pick near it.

<br>

#### Q6: "How would I make reads genuinely concurrent (a real `shared_mutex`)?"

Redesign `get` to not mutate shared state: use **Clock/second-chance** (set an atomic referenced bit, defer the real ordering to eviction), per-entry access timestamps, or batched recency logs drained by a background thread (Caffeine's approach). All trade exact LRU for approximate LRU in exchange for lock-free / shared reads.

<br>

#### Q7: "Why must a thread-safe getter return by value?"

If it returned a reference into the cache, the lock would be released when the function returns, but the caller would hold a reference to data another thread can evict or overwrite → dangling reference / data race. Return a copy (`std::optional<V>`) so the caller owns stable data once the lock is dropped.

<br>

#### Q8: "What is false sharing and where does it bite a sharded cache?"

False sharing is when independent variables sit on the same CPU cache line, so writing one invalidates the other's cache copy across cores. Per-shard hit/miss counters (or the shard mutexes themselves) packed tightly in an array can falsely share, silently killing the scaling sharding was meant to provide. Fix: pad/align each shard's mutable state to a cache line (`alignas(64)`).

<br><br>

---

## Reflection Questions

1. Why is `get` in an exact LRU cache a *write* operation, and why does that make a `shared_mutex` read lock incorrect?
2. What are the pros and cons of a single coarse mutex vs sharding?
3. Why must shard count be a power of two for the masking trick, and how do you pick the count?
4. What exactness does sharding sacrifice, and why is it usually acceptable?
5. How do Clock / batched-recency approaches recover concurrent reads, and what do they trade away?
6. Why must thread-safe getters return by value, and where does false sharing threaten a sharded cache?

---

## Interview Questions

1. "Make your LRU cache thread-safe. What's the simplest correct approach?"
2. "Why is using a `shared_mutex` for `get` a bug in an LRU cache?"
3. "Explain sharding / segmented locking. How much does it improve scalability?"
4. "What correctness does a sharded LRU give up compared to a single global LRU?"
5. "How would you choose the number of shards?"
6. "How could you make reads truly concurrent? Describe approximate-LRU schemes."
7. "Why can't a thread-safe getter return a reference into the cache?"
8. "What is false sharing and how does it affect a sharded cache's counters?"
9. "Sharding doesn't help with one very hot key — why, and what would you do?"
10. "Compare exact LRU (splice) with Clock/second-chance for a concurrent cache."

---

**Next**: Day 40 — Design a Rate Limiter →
