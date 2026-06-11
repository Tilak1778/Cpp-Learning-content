# Day 55: Mock Interview #3

[← Back to Study Plan](../lld-study-plan.md) | [← Day 54](day54-mock-interview-2.md)

> **Time**: ~2-3 hours (weekend day)
> **Goal**: Final mock — the longest and most realistic. Today's set: **Key-Value Store** (Day 44, `day44-kv-store.md`), **Memory Allocator** (Days 9-10, `day09-arena-allocator.md` / `day10-pool-allocator.md`), and **Connection Pool** (Day 43, `day43-connection-pool.md`). The worked example is the **KV Store**. The emphasis this round is **full simulation and adapting the design live to follow-up "what-ifs."** Real senior LLD rounds are a *conversation*: you build a v1, the interviewer changes a requirement ("now it must persist," "now it's 100× the data," "now it's concurrent"), and you evolve the design without throwing it away. Practice the evolution, not just the initial answer.

---
---

# PART 1: HOW TO RUN A FULL SIMULATION

---
---

<br>

<h2 style="color: #2980B9;">📘 55.1 The Extended Clock</h2>

Because this is a weekend day, run a longer, more realistic loop and do it twice (two problems back-to-back, or one problem with deep follow-ups):

```
 0:00 ─ 0:05   Clarify
 0:05 ─ 0:20   Design v1 on paper
 0:20 ─ 0:50   Implement v1
 0:50 ─ 1:10   Live evolution: interviewer changes a requirement; adapt the design
 1:10 ─ 1:15   Trace + recap the evolution path
```

The crucial new skill is **0:50–1:10**: taking a working v1 and *extending* it under a new constraint without a rewrite. Senior candidates are judged on whether their v1 design has *seams* the new requirement can slot into. A monolithic v1 forces a rewrite; a layered v1 absorbs the change.

<br>

<h2 style="color: #2980B9;">📘 55.2 Designing for Evolution</h2>

Build v1 so the predictable follow-ups have a place to land:

| Likely follow-up | Seam to leave in v1 |
|------------------|---------------------|
| "Now make it thread-safe" | Keep mutation behind a single method you can lock; don't expose raw internals |
| "Now make it persist" | Put storage behind an interface; in-memory is one impl |
| "Now 100× the data" | Keep the data structure swappable (map → LSM/B-tree); state the complexity |
| "Now it's distributed" | Key access goes through one function you can shard/hash |

State this intent aloud during v1: *"I'll hide the backing store behind an interface so that if we later need persistence, it's a new implementation, not a rewrite."* That single sentence signals seniority.

<br>

<h2 style="color: #2980B9;">📘 55.3 The Self-Simulation Protocol (solo practice)</h2>

Without a human interviewer, *be* the interviewer between phases. After v1 compiles, draw a "what-if" card and adapt:

1. Set a timer for the full loop and narrate aloud (record yourself if possible).
2. At 0:50, pick one follow-up from Part 4 *at random* — don't pre-plan which.
3. Adapt the design *live*, editing the v1 code, not starting over.
4. Score with the rubric (Part 3), paying attention to the new **adaptability** axis.

<br><br>

---
---

# PART 2: WORKED EXAMPLE — KEY-VALUE STORE

---
---

<br>

<h2 style="color: #2980B9;">📘 55.4 Step 1 — Clarify (≈5 min)</h2>

Prompt: *"Design an in-memory key-value store."*

| Question | Assumed answer |
|----------|----------------|
| Operations? | `get`, `put`, `del`, `exists`. (Maybe range scan — ask.) |
| Value type? | Opaque `std::string` (bytes) for v1. |
| Ordered keys or hash? | Unordered (point lookups dominate); mention ordered if range scans needed. |
| Persistence? | In-memory v1; expect "make it durable" as a follow-up. |
| Concurrency? | Single-threaded v1; expect "make it concurrent." |
| Eviction / size cap? | Unbounded v1; expect "add TTL/LRU." |

Note how you *pre-name the follow-ups* — that's designing for evolution before a line of code.

<br>

<h2 style="color: #2980B9;">📘 55.5 Step 2 — Design v1 (≈15 min)</h2>

**API behind an interface (the key seam):**

```cpp
class IKvStore {
public:
    virtual ~IKvStore() = default;
    virtual std::optional<std::string> get(const std::string& k) const = 0;
    virtual void put(const std::string& k, std::string v)             = 0;
    virtual bool del(const std::string& k)                            = 0;
    virtual bool exists(const std::string& k) const                   = 0;
};
```

> Defining `IKvStore` up front is deliberate: when "make it persistent" arrives, a `PersistentKvStore` is a *new implementation*, and the in-memory one becomes a cache layer in front of it — no caller changes.

**v1 implementation — `unordered_map`:**

```cpp
#include <optional>
#include <string>
#include <unordered_map>

class MemoryKvStore : public IKvStore {
    std::unordered_map<std::string, std::string> data_;
public:
    std::optional<std::string> get(const std::string& k) const override {
        if (auto it = data_.find(k); it != data_.end()) return it->second;
        return std::nullopt;
    }
    void put(const std::string& k, std::string v) override {
        data_[k] = std::move(v);                       // insert or overwrite
    }
    bool del(const std::string& k) override {
        return data_.erase(k) > 0;
    }
    bool exists(const std::string& k) const override {
        return data_.find(k) != data_.end();
    }
};
```

Articulate: *"`unordered_map` gives O(1) average get/put/del — perfect for point lookups. If the interviewer wants ordered range scans I'd use `std::map` (O(log n)) or, at scale, a B-tree / LSM-tree."*

<br>

<h2 style="color: #2980B9;">📘 55.6 Step 3 — Live Evolution (≈20 min)</h2>

Now play out the evolution sequence the interviewer is likely to drive. Each step *extends* v1.

**Evolution A — "Make it thread-safe."**

Wrap mutations; use a `shared_mutex` since reads dominate:

```cpp
#include <shared_mutex>

class ConcurrentKvStore : public IKvStore {
    mutable std::shared_mutex mtx_;
    std::unordered_map<std::string, std::string> data_;
public:
    std::optional<std::string> get(const std::string& k) const override {
        std::shared_lock lk(mtx_);                     // many concurrent readers
        if (auto it = data_.find(k); it != data_.end()) return it->second;
        return std::nullopt;
    }
    void put(const std::string& k, std::string v) override {
        std::unique_lock lk(mtx_);                     // exclusive writer
        data_[k] = std::move(v);
    }
    bool del(const std::string& k) override {
        std::unique_lock lk(mtx_); return data_.erase(k) > 0;
    }
    bool exists(const std::string& k) const override {
        std::shared_lock lk(mtx_); return data_.count(k) > 0;
    }
};
```

Say: *"`shared_mutex` lets reads run concurrently; writes are exclusive. If writes were frequent or contention high, I'd **shard** — N buckets each with its own map+lock, keyed by `hash(k) % N` — turning one global lock into N independent ones."*

**Evolution B — "Make it durable / persistent."**

Add a **write-ahead log (WAL)**: append every mutation to an append-only file *before* applying it in memory; on startup, replay the log to rebuild state. This is the classic durability pattern (and the seam from §55.5 pays off — it's a new impl, not a rewrite).

```cpp
void put(const std::string& k, std::string v) override {
    wal_.append("PUT " + k + " " + v + "\n");          // durable first
    std::unique_lock lk(mtx_);
    data_[k] = std::move(v);                            // then memory
}
```

Trade-offs to state: append-only WAL gives durability with sequential (fast) writes; to bound replay time, periodically **snapshot** the map and truncate the log (compaction). For real scale this is exactly an **LSM-tree** (memtable + WAL + SSTables) — name-drop it.

**Evolution C — "Add TTL / eviction."**

Store `(value, expiry)`; treat expired entries as misses (lazy eviction) plus a background sweeper or a min-heap by expiry (proactive). For a size cap, layer the **LRU** design from Day 53/Day 38 on top — the interface stays the same.

<br>

<h2 style="color: #2980B9;">📘 55.7 The Evolution Map (recap aloud at the end)</h2>

```
   v1: unordered_map               ── O(1) point ops, single-threaded
    │
    ├─ +shared_mutex / sharding    ── concurrent
    │
    ├─ +WAL +snapshot/compaction   ── durable  →  (at scale: LSM-tree)
    │
    └─ +TTL / +LRU cap             ── bounded
```

Recapping the path shows the interviewer you understand *how a real KV store grows from a hash map into a storage engine* — and that your v1 interface absorbed every change.

<br><br>

---
---

# PART 3: SELF-EVALUATION RUBRIC

---
---

<br>

<h2 style="color: #2980B9;">📘 55.8 Score Yourself (with an Adaptability axis)</h2>

Same 0–4 scale; today add a sixth axis — **adaptability** — since live evolution is the focus.

| Axis | Strong (3–4) this round |
|------|-------------------------|
| **Correctness** | v1 right; each evolution preserves invariants (WAL before memory, etc.) |
| **API design** | Interface seam (`IKvStore`) defined first; callers unchanged across evolutions |
| **Concurrency safety** | `shared_mutex` for read-heavy; sharding named; WAL append ordered before apply |
| **Communication** | Pre-named follow-ups; narrated each evolution as an *extension* |
| **Complexity analysis** | O(1) hash ops; O(log n) if ordered; replay/compaction cost discussed |
| **Adaptability (new)** | Absorbed ≥2 requirement changes *without* rewriting v1 |

**Target: ≥ 3 average, and ≥ 3 on adaptability.** A v1 that needed a full rewrite when "make it concurrent" arrived scores low here regardless of how clean v1 was.

<br>

<h2 style="color: #2980B9;">📘 55.9 Senior Signals Checklist</h2>

Tick these off after the mock:

- [ ] Defined an interface/seam before the concrete v1.
- [ ] Pre-named the likely follow-ups during clarification.
- [ ] Each "what-if" was an *extension*, not a rewrite.
- [ ] Named the production-scale endpoint (LSM-tree, slab allocator, connection-pool health checks).
- [ ] Stated lock granularity and the sharding escape hatch.
- [ ] Recapped the evolution path at the end.

<br><br>

---
---

# PART 4: FOLLOW-UP "WHAT-IF" QUESTIONS

---
---

<br>

<h2 style="color: #2980B9;">📘 55.10 KV Store Follow-ups</h2>

#### "Now keys must support range scans (`scan(a, b)`)."

Switch the backing store from `unordered_map` to an ordered structure: `std::map` (O(log n), red-black tree) for moderate size, or a **B-tree / LSM-tree** at scale. The `IKvStore` interface gains a `scan` method; existing point ops still work. Trade-off: ordered structures are O(log n) per op vs O(1), and worse cache behavior — justified only if range queries are needed.

#### "100× the data — it doesn't fit in RAM."

Move to an **LSM-tree**: writes go to an in-memory memtable + WAL; when the memtable fills, flush it as an immutable sorted SSTable on disk; background compaction merges SSTables. Reads check memtable, then SSTables newest-to-oldest (with bloom filters to skip files). This is RocksDB/LevelDB. Trade-off: write-optimized, read amplification mitigated by bloom filters and compaction.

#### "Two clients write the same key concurrently — last-writer-wins is wrong."

Add **versioning / optimistic concurrency**: `put` takes an expected version (CAS — compare-and-set); reject if the stored version differs, forcing the client to re-read and retry. Or use MVCC (keep versioned values). State that this moves conflict resolution to the application.

<br>

<h2 style="color: #2980B9;">📘 55.11 Memory Allocator Follow-ups (problem 2)</h2>

Design recap: an **arena/bump allocator** (Day 9) hands out memory by advancing a pointer — O(1) alloc, no per-object free (reset the whole arena at once). A **pool/slab allocator** (Day 10) manages fixed-size blocks via a free list — O(1) alloc and free, no fragmentation for that size class. (See `day09-arena-allocator.md`, `day10-pool-allocator.md`.)

#### "Arena vs pool — when each?"

Arena: many objects with a *shared lifetime* (per-request, per-frame) freed together — fastest, zero per-object free. Pool: many *individually* allocated/freed objects of the *same size* (nodes, particles) — O(1) free via free list, reused immediately. Use arena for phase-scoped allocation, pool for object churn.

#### "Make the pool allocator thread-safe / scalable."

A single mutex on the free list serializes all allocs. Better: **per-thread free lists** (thread-local pools) that batch-refill from a global pool under a lock occasionally — exactly how tcmalloc/jemalloc work. Lock-free Treiber-stack free lists are another option. Trade-off: per-thread caches use more memory but eliminate contention.

#### "Variable-sized allocations, not one size class."

Use **multiple size classes** (e.g., 16, 32, 64, ... bytes) — round each request up to the nearest class and serve from that class's pool. This is the slab/segregated-fit design. Trade-off: internal fragmentation (rounding waste) vs O(1) speed and no external fragmentation.

#### "How do you detect use-after-free in the pool?"

Poison freed blocks with a sentinel pattern (e.g., `0xDEADBEEF`) and check on realloc; maintain a free-bit per block to detect double-free. Mention ASan does this in production builds.

<br>

<h2 style="color: #2980B9;">📘 55.12 Connection Pool Follow-ups (problem 3)</h2>

Design recap: pre-create expensive connections; `acquire()` hands out an idle one (or grows up to a max), `release()` returns it; RAII handle ensures release; block-with-timeout when exhausted. This is the Object Pool pattern (Day 13) specialized to connections. (See `day43-connection-pool.md`.)

#### "All connections are in use and a new request arrives."

Options, with trade-offs: (a) **block** on a condition variable until one is released, with a timeout → backpressure but bounded waiting; (b) **grow** up to a hard max then block/fail; (c) **reject** immediately → fail fast. Production pools usually do grow-to-max-then-block-with-timeout.

#### "A pooled connection died (server dropped it) while idle."

Add **health checks**: validate (ping / test query) on `acquire`, or run a background reaper that pings idle connections and evicts dead ones. Trade-off: validate-on-acquire adds latency to every checkout; background reaping adds a thread but keeps checkout fast. Also implement **max-lifetime** to recycle connections proactively before the DB times them out.

#### "Connections leak — callers forget to release."

Return an **RAII handle** (Day 13's `PoolHandle`) whose destructor releases automatically, even on exception/early return. As a safety net, track checkout timestamps and log/reclaim connections held longer than a threshold (leak detection).

#### "Make `acquire` fair under contention."

A plain `condition_variable` can starve waiters. Use a FIFO wait queue (ticket/turnstile) so the longest-waiting requester gets the next released connection. Trade-off: fairness vs slightly higher overhead than a free-for-all notify.

<br><br>

---

## Reflection Questions

1. Did your KV v1 absorb "make it concurrent" and "make it durable" *without* a rewrite? If not, what seam was missing?
2. Why write to the WAL *before* applying to memory? What durability guarantee breaks if you reverse the order?
3. When is an arena allocator the right tool and when is a pool allocator, in terms of object lifetimes?
4. For each problem, what is the production-scale endpoint you'd name (LSM-tree / per-thread caches / health-checked pool)?
5. How does sharding recur as the answer to lock contention across the KV store, the allocator, and (implicitly) the connection pool?
6. Across all three mocks (Days 53–55), which rubric axis is consistently your weakest? What is your concrete plan to fix it before the real interview?

---

## Interview Questions

1. "Design an in-memory KV store, then evolve it to be concurrent, durable, and bounded."
2. "Why put the backing store behind an interface from the start?"
3. "Explain a write-ahead log and how snapshot + compaction bound replay time."
4. "When would you switch a KV store from a hash map to an LSM-tree?"
5. "How do you handle concurrent writes to the same key beyond last-writer-wins?"
6. "Arena vs pool allocator — design each and say when you'd use which."
7. "Make a pool allocator scale across many threads without a global lock."
8. "Design a connection pool. What happens when it's exhausted?"
9. "How do you detect and recover from dead pooled connections?"
10. "How do you prevent and detect connection leaks in a pool?"

---

**Next**: Day 56 — Final Review & Cheat Sheet →
