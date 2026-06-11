# Day 38: Design an LRU Cache

[← Back to Study Plan](../lld-study-plan.md) | [← Day 37](day37-logger-async.md)

> **Time**: ~1.5-2 hours
> **Goal**: The LRU (Least-Recently-Used) cache is the single most-asked data-structure design question in interviews, and the canonical answer — a **hash map** of keys plus an **intrusive doubly-linked list** ordering entries by recency — is a beautiful lesson in combining structures to hit **O(1) for both `get` and `put`**. Today you learn *why* a list and *why* a map, the **splice trick** that moves a node to the front in O(1), how **eviction** picks the tail, and you build a generic `LRUCache<K,V>` with `get`, `put`, an eviction callback, and stats.

---
---

# PART 1: THE PROBLEM

---
---

<br>

<h2 style="color: #2980B9;">📘 38.1 What an LRU Cache Must Do</h2>

A cache holds a bounded number of key→value entries. When it's full and a new key arrives, it must **evict** one existing entry. *Which* one? LRU's answer: **the one untouched for the longest time** — the least recently used.

The contract:

| Operation | Semantics | Required complexity |
|-----------|-----------|---------------------|
| `get(k)` | Return value if present; mark `k` as *most recently used* | **O(1)** |
| `put(k, v)` | Insert/update; mark `k` MRU; if over capacity, evict the LRU entry | **O(1)** |

The "mark as most recently used" on *every access* is the crux. A read isn't passive — it reorders recency. That reordering is what makes the naive implementations too slow.

<br>

#### Why LRU (the intuition)

LRU bets on **temporal locality**: data used recently is likely to be used again soon (loop variables, hot DB rows, recently-viewed pages). Evicting the least-recently-used entry discards the coldest data. It's the default eviction policy for OS page caches, CPU caches (approximately), database buffer pools, and CDN edge caches.

```
Access order (newest → oldest):     D   C   B   A
                                     ▲               ▲
                                MRU (keep)      LRU (evict next)
```

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 38.2 Why Naive Approaches Fail</h2>

Before the right answer, see why the obvious ones are too slow:

| Approach | `get` recency update | `put` eviction | Verdict |
|----------|----------------------|----------------|---------|
| **Array, scan for key** | O(n) find + O(n) shift to front | O(n) find min timestamp | O(n) — too slow |
| **`map` + timestamp field** | O(1) find, but evicting LRU = O(n) scan for min ts | O(n) | O(n) eviction |
| **`std::list` only** | O(n) to find the key in the list | O(1) pop_back | O(n) lookup |
| **`unordered_map` + `list`** | O(1) find + **O(1) splice** to front | O(1) pop tail | **O(1) both** ✅ |

The winning combination uses each structure for what it's *good* at:
- **Hash map** → O(1) *lookup by key*.
- **Doubly-linked list** → O(1) *reordering* (move a node, remove a node) given a pointer to the node.

Neither alone is enough. Together they're O(1) for everything.

<br><br>

---
---

# PART 2: THE HASH-MAP + DOUBLY-LINKED-LIST DESIGN

---
---

<br>

<h2 style="color: #2980B9;">📘 38.3 The Layout</h2>

```
   Doubly-linked list, ordered by recency (front = MRU, back = LRU):

   HEAD ⇄ [K=3,V=c] ⇄ [K=1,V=a] ⇄ [K=2,V=b] ⇄ TAIL
            (MRU)                     (LRU → evicted next)

   Hash map: key → iterator/pointer INTO the list node

   ┌─────┬───────────────────────┐
   │  3  │ ─► node[K=3,V=c]       │
   │  1  │ ─► node[K=1,V=a]       │
   │  2  │ ─► node[K=2,V=b]       │
   └─────┴───────────────────────┘
```

Each list node stores **both** the key and the value. The map maps a key to the *position* of its node in the list. Why store the key in the node too? Because on eviction you pop the tail node and need its key to erase it from the map — the node must know its own key.

<br>

#### The two operations, mechanically

**`get(k)`:**
1. Look up `k` in the map → O(1). Miss → return `nullopt`.
2. **Splice** that node to the front of the list (now MRU) → O(1).
3. Return the value.

**`put(k, v)`:**
1. If `k` exists: update its value, splice node to front → O(1). Done.
2. Else: push a new `[k,v]` node at the front; insert `k → front` into the map → O(1).
3. If `size() > capacity`: take the **tail** node, erase its key from the map, pop it from the list (evict) → O(1).

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 38.4 The Splice Trick</h2>

The whole design hinges on one `std::list` member function:

```cpp
list.splice(list.begin(), list, it);   // move node `it` to the front, O(1), NO copy
```

`splice` **relinks pointers** — it does not move or copy the element. The node's address is unchanged, so the iterator (`it`) and any pointers into it stay valid. That's why we can keep iterators in the map: a splice doesn't invalidate them.

```
Before:  HEAD ⇄ A ⇄ B ⇄ [C] ⇄ TAIL        splice C to front:
                       ↑ it (the node we touched)

After:   HEAD ⇄ [C] ⇄ A ⇄ B ⇄ TAIL        only 6 pointers rewired, O(1)
```

> **`std::list` iterator stability is the magic.** Per the standard, `std::list` insert/splice/erase invalidate iterators *only to the erased element*. Splicing C to the front does **not** invalidate the iterators to A, B, or even C. This is precisely why we can store `std::list<...>::iterator` values inside an `unordered_map` and trust them across reorderings. (You could *not* do this with `std::vector`, whose iterators invalidate on reallocation.)

<br>

#### Doing it by hand (the intrusive list interview wants)

In an interview you're often expected to hand-roll the doubly-linked list (no `std::list`), because it shows you understand the pointer surgery and because it's faster (one allocation per node, no `std::list` overhead). The move-to-front primitive:

```cpp
struct Node {
    K key; V val;
    Node* prev = nullptr;
    Node* next = nullptr;
};

// Detach a node from wherever it is (O(1)).
void unlink(Node* n) {
    n->prev->next = n->next;
    n->next->prev = n->prev;
}

// Insert n right after the sentinel head (front = MRU).
void insertFront(Node* n) {
    n->next        = head->next;
    n->prev        = head;
    head->next->prev = n;
    head->next       = n;
}

void moveToFront(Node* n) { unlink(n); insertFront(n); }
```

Using **sentinel** head and tail dummy nodes removes all the `if (n == head)` / `if (n == tail)` null-checks — every real node always has a non-null `prev` and `next`. This is the single biggest source of bugs in hand-rolled linked lists, and sentinels eliminate it.

```
   [HEAD sentinel] ⇄ realA ⇄ realB ⇄ [TAIL sentinel]
        ▲                                   ▲
   never holds data;                  evict = tail->prev
   front = head->next
```

<br><br>

---
---

# PART 3: IMPLEMENTATION

---
---

<br>

<h2 style="color: #2980B9;">📘 38.5 The `std::list` Version (Clean & Idiomatic)</h2>

```cpp
#include <list>
#include <unordered_map>
#include <optional>
#include <functional>
#include <utility>

template <typename K, typename V>
class LRUCache {
    using Entry = std::pair<K, V>;                 // node stores key AND value
    using ListIt = typename std::list<Entry>::iterator;

    std::size_t                       m_cap;
    std::list<Entry>                  m_list;       // front = MRU, back = LRU
    std::unordered_map<K, ListIt>     m_map;        // key → node position
    std::function<void(const K&, const V&)> m_onEvict;

public:
    explicit LRUCache(std::size_t capacity,
                      std::function<void(const K&, const V&)> onEvict = {})
        : m_cap(capacity), m_onEvict(std::move(onEvict)) {
        // capacity 0 would make put() evict immediately; guard if you wish.
    }

    std::optional<V> get(const K& key) {
        auto it = m_map.find(key);
        if (it == m_map.end()) return std::nullopt;     // miss
        m_list.splice(m_list.begin(), m_list, it->second);  // O(1) move to front
        return it->second->second;                       // value
    }

    void put(const K& key, V value) {
        auto it = m_map.find(key);
        if (it != m_map.end()) {                         // update existing
            it->second->second = std::move(value);
            m_list.splice(m_list.begin(), m_list, it->second);
            return;
        }
        m_list.emplace_front(key, std::move(value));     // new MRU node
        m_map[key] = m_list.begin();

        if (m_map.size() > m_cap) evictLRU();
    }

    bool contains(const K& key) const { return m_map.count(key) > 0; }
    std::size_t size() const { return m_map.size(); }
    std::size_t capacity() const { return m_cap; }

private:
    void evictLRU() {
        auto& victim = m_list.back();                    // LRU = tail
        if (m_onEvict) m_onEvict(victim.first, victim.second);
        m_map.erase(victim.first);
        m_list.pop_back();
    }
};
```

<br>

#### Walkthrough of the subtle lines

- `m_list.splice(m_list.begin(), m_list, it->second)` — moves the node `it->second` to the front of *the same* list. The iterator stays valid, so `m_map` need not be updated.
- `it->second->second` — `it->second` is the **list iterator**; dereferencing gives the `Entry` (`pair`); `.second` is the value. (The double `.second` trips everyone up once.)
- On eviction, we read the victim's **key** from the node (`victim.first`) to erase the right map entry. This is *why the node stores the key*.
- The eviction check is `> m_cap` (strictly greater): we insert first, then trim if we overshot by one.

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 38.6 The Hand-Rolled Version (Interview Style)</h2>

If asked to avoid `std::list`, here's the intrusive-list version with sentinels:

```cpp
#include <unordered_map>

template <typename K, typename V>
class LRUCacheManual {
    struct Node {
        K key{}; V val{};
        Node* prev = nullptr;
        Node* next = nullptr;
    };

    std::size_t                  m_cap;
    Node*                        m_head;   // sentinel; m_head->next = MRU
    Node*                        m_tail;   // sentinel; m_tail->prev = LRU
    std::unordered_map<K, Node*> m_map;

    void unlink(Node* n) {
        n->prev->next = n->next;
        n->next->prev = n->prev;
    }
    void insertFront(Node* n) {
        n->next = m_head->next;
        n->prev = m_head;
        m_head->next->prev = n;
        m_head->next = n;
    }
    void moveToFront(Node* n) { unlink(n); insertFront(n); }

public:
    explicit LRUCacheManual(std::size_t cap) : m_cap(cap) {
        m_head = new Node();
        m_tail = new Node();
        m_head->next = m_tail;
        m_tail->prev = m_head;
    }
    ~LRUCacheManual() {
        Node* n = m_head;
        while (n) { Node* nx = n->next; delete n; n = nx; }
    }
    LRUCacheManual(const LRUCacheManual&) = delete;
    LRUCacheManual& operator=(const LRUCacheManual&) = delete;

    bool get(const K& key, V& out) {
        auto it = m_map.find(key);
        if (it == m_map.end()) return false;
        moveToFront(it->second);
        out = it->second->val;
        return true;
    }

    void put(const K& key, const V& value) {
        auto it = m_map.find(key);
        if (it != m_map.end()) {
            it->second->val = value;
            moveToFront(it->second);
            return;
        }
        Node* n = new Node{key, value, nullptr, nullptr};
        insertFront(n);
        m_map[key] = n;

        if (m_map.size() > m_cap) {
            Node* victim = m_tail->prev;     // LRU
            unlink(victim);
            m_map.erase(victim->key);
            delete victim;
        }
    }
    std::size_t size() const { return m_map.size(); }
};
```

The sentinels (`m_head`, `m_tail`) mean `unlink`/`insertFront` never touch null — every real node always sits between two non-null neighbors. The destructor walks the whole chain (including sentinels) to free everything.

> **Why the `std::list` version is better in production:** it's shorter, exception-safe (no manual `new`/`delete`), and just as fast (`std::list` nodes are also individually allocated). Hand-rolling is an *interview* exercise to prove you understand the pointers. Reach for `std::list` in real code.

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 38.7 Eviction Callbacks</h2>

Real caches need to *react* to eviction: close the evicted DB connection, write a dirty page back to disk, decrement a refcount, fire a metric. That's the **eviction callback**:

```cpp
LRUCache<int, std::string> cache(2, [](const int& k, const std::string& v) {
    std::cout << "evicting " << k << " => " << v << "\n";
});
```

A few correctness rules for the callback:

- Fire it **before** removing the entry from the map/list, while the data is still valid.
- The callback runs *inside* `put()` — keep it fast and **don't re-enter the cache** from it (re-entrancy can invalidate the iterator/node you're mid-evicting → UB).
- In a thread-safe cache (Day 39), decide carefully whether the callback runs **under the lock** (simple but risks deadlock if it calls back) or after releasing it (safer, but the entry is already gone).

<br><br>

---
---

# PART 4: VARIANTS & TRADE-OFFS

---
---

<br>

<h2 style="color: #2980B9;">📘 38.8 LRU vs Other Eviction Policies</h2>

| Policy | Evicts | Strength | Weakness |
|--------|--------|----------|----------|
| **LRU** | Least recently *used* | Great for temporal locality | A single big scan can evict the whole hot set ("cache pollution") |
| **LFU** | Least *frequently* used | Keeps genuinely popular items | Slow to adapt; needs frequency counts + tie-breaking; "cache stale popularity" |
| **FIFO** | Oldest inserted | Trivial (a queue) | Ignores usage — evicts hot items that happen to be old |
| **MRU** | *Most* recently used | Good for scan-once workloads | Counterintuitive; rarely the default |
| **Random** | A random entry | O(1), no metadata, no bookkeeping | No locality awareness |
| **2Q / LRU-K / ARC** | Hybrid | Resists scan pollution; adaptive | More complex; more state |

LRU is the sensible default. When a sequential scan pollutes it, production systems reach for **scan-resistant** variants (ARC in ZFS, 2Q, LRU-K) that protect frequently-used items from one-time bulk accesses.

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 38.9 Common Bugs & Edge Cases</h2>

#### Bug 1: Forgetting to update recency on `get`

```cpp
V get(const K& k) { return m_map.at(k)->second; }   // BUG: no splice → recency never changes
```

Without the splice, your "LRU" cache degenerates to FIFO. `get` *must* move the node to the front.

<br>

#### Bug 2: Storing only the value in the list node

If the node holds just the value, eviction can't find the key to erase from the map. Store **both** `key` and `value` in the node.

<br>

#### Bug 3: Using `std::vector` iterators in the map

`std::vector` iterators invalidate on reallocation. Only `std::list` (and `std::map`/node-based containers) give the iterator stability this design relies on. Don't substitute a vector for the recency list.

<br>

#### Bug 4: Capacity-0 cache

With `capacity == 0`, every `put` inserts then immediately evicts the just-inserted entry. Decide the semantics (reject? store nothing?) and guard explicitly.

<br>

#### Bug 5: Update-then-evict ordering

On `put` of a *new* key into a full cache, insert first, *then* evict. If you evict first based on stale size, you can wrongly evict when updating an existing key (which shouldn't grow the cache). Check existence before deciding to evict.

<br><br>

---
---

# PART 5: BUILD EXERCISE

---
---

<br>

<h2 style="color: #2980B9;">📘 38.10 Exercise: A Generic O(1) LRU Cache</h2>

Build `LRUCache<K,V>` with `get`, `put`, capacity-based eviction, an eviction callback, and hit/miss stats. Verify the eviction order with asserts.

<br>

#### Skeleton

```cpp
// lru_cache.h
#pragma once
#include <cstddef>
#include <functional>
#include <list>
#include <optional>
#include <unordered_map>
#include <utility>

template <typename K, typename V>
class LRUCache {
    using Entry  = std::pair<K, V>;
    using ListIt = typename std::list<Entry>::iterator;

    std::size_t                             m_cap;
    std::list<Entry>                        m_list;   // front = MRU, back = LRU
    std::unordered_map<K, ListIt>           m_map;
    std::function<void(const K&, const V&)> m_onEvict;

    std::size_t m_hits = 0, m_misses = 0, m_evictions = 0;

public:
    explicit LRUCache(std::size_t capacity,
                      std::function<void(const K&, const V&)> onEvict = {})
        : m_cap(capacity), m_onEvict(std::move(onEvict)) {}

    std::optional<V> get(const K& key) {
        auto it = m_map.find(key);
        if (it == m_map.end()) { ++m_misses; return std::nullopt; }
        ++m_hits;
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
        if (m_map.size() > m_cap) evictLRU();
    }

    bool        contains(const K& key) const { return m_map.count(key) > 0; }
    std::size_t size() const                 { return m_map.size(); }
    std::size_t capacity() const             { return m_cap; }
    std::size_t hits() const                 { return m_hits; }
    std::size_t misses() const               { return m_misses; }
    std::size_t evictions() const            { return m_evictions; }

private:
    void evictLRU() {
        auto& victim = m_list.back();
        if (m_onEvict) m_onEvict(victim.first, victim.second);
        m_map.erase(victim.first);
        m_list.pop_back();
        ++m_evictions;
    }
};
```

<br>

#### Test driver

```cpp
// main.cpp
#include "lru_cache.h"
#include <cassert>
#include <iostream>
#include <vector>

int main() {
    // 1. Basic get/put + eviction order
    {
        std::vector<int> evicted;
        LRUCache<int, std::string> c(2,
            [&](const int& k, const std::string&) { evicted.push_back(k); });

        c.put(1, "a");
        c.put(2, "b");                       // cache: [2,1]   (MRU..LRU)
        assert(c.get(1).value() == "a");     // touch 1 → cache: [1,2]
        c.put(3, "c");                       // over cap → evict LRU = 2; cache: [3,1]
        assert(!c.contains(2));
        assert(evicted.size() == 1 && evicted[0] == 2);
        assert(c.get(2) == std::nullopt);    // 2 was evicted
        assert(c.get(3).value() == "c");
        assert(c.get(1).value() == "a");
        std::cout << "[1] eviction order OK\n";
    }

    // 2. Update existing key does NOT grow size or evict
    {
        LRUCache<int, int> c(2);
        c.put(1, 10);
        c.put(2, 20);
        c.put(1, 11);                        // update, not insert
        assert(c.size() == 2);
        assert(c.evictions() == 0);
        assert(c.get(1).value() == 11);
        assert(c.get(2).value() == 20);      // 2 still present
        std::cout << "[2] update-not-grow OK\n";
    }

    // 3. get() updates recency (proves it's LRU, not FIFO)
    {
        LRUCache<int, int> c(2);
        c.put(1, 1);
        c.put(2, 2);
        c.get(1);                            // touch 1 → 2 is now LRU
        c.put(3, 3);                         // evicts 2 (not 1)
        assert(c.contains(1));               // survived because of the get()
        assert(!c.contains(2));
        std::cout << "[3] get() updates recency OK\n";
    }

    // 4. Stats
    {
        LRUCache<int, int> c(2);
        c.put(1, 1);
        c.get(1);                            // hit
        c.get(99);                           // miss
        assert(c.hits() == 1 && c.misses() == 1);
        std::cout << "[4] stats OK\n";
    }

    // 5. Eviction callback fires with correct value
    {
        std::string last;
        LRUCache<int, std::string> c(1,
            [&](const int&, const std::string& v) { last = v; });
        c.put(1, "first");
        c.put(2, "second");                  // evicts (1,"first")
        assert(last == "first");
        std::cout << "[5] eviction callback OK\n";
    }

    std::cout << "All assertions passed.\n";
    return 0;
}
```

<br>

#### Compile and run

```bash
g++ -std=c++17 -Wall -Wextra -Wpedantic -fsanitize=address \
    -o day38 main.cpp && ./day38
```

<br>

#### Expected output

```
[1] eviction order OK
[2] update-not-grow OK
[3] get() updates recency OK
[4] stats OK
[5] eviction callback OK
All assertions passed.
```

<br>

#### Bonus Challenges

1. **Hand-rolled list** — re-implement using the intrusive doubly-linked list with sentinels from §38.6 (no `std::list`). Run under ASan to prove there are no leaks or use-after-free.

2. **TTL (time-aware) cache** — add an expiry per entry; `get` returns `nullopt` for an expired entry and removes it. Decide: lazy expiry (on access) vs a background sweeper.

3. **`peek(k)`** — a read that returns the value *without* updating recency (useful for diagnostics/metrics). Compare its effect on hit ratio in a workload.

4. **LFU variant** — replace the recency list with a frequency structure (the classic O(1) LFU uses a list-of-frequency-buckets, each a list of equal-frequency nodes). Compare hit ratios on a skewed (Zipfian) access trace.

5. **Resize at runtime** — add `setCapacity(n)`; if shrinking, evict from the tail until `size() <= n`, firing the callback for each.

6. **Hit-ratio benchmark** — feed a Zipfian key distribution and plot hit ratio vs capacity for LRU vs FIFO vs Random. Show LRU's advantage and the scan-pollution failure mode.

<br><br>

---
---

# PART 6: DEEP DIVE — COMMON QUESTIONS

---
---

<br>

<h2 style="color: #2980B9;">📘 38.11 Q&A</h2>

<br>

#### Q1: "Why both a hash map and a linked list?"

Each does one thing in O(1) that the other can't. The hash map gives O(1) **lookup by key**. The doubly-linked list gives O(1) **reordering** (move-to-front) and O(1) **eviction** (pop tail) — but only if you already have a pointer to the node, which the map provides. Either structure alone forces an O(n) operation; together they're O(1) everywhere.

<br>

#### Q2: "What is the splice trick and why is it O(1)?"

`std::list::splice` relinks node pointers to move a node to a new position without copying or moving the element. Moving one node is a constant number of pointer assignments — O(1). Crucially, splice does **not** invalidate the node's iterator, so the iterators stored in the map remain valid across reorderings.

<br>

#### Q3: "Why must the list node store the key, not just the value?"

On eviction you pop the **tail** node and must remove its entry from the map. To `erase` from the map you need the **key**, which therefore has to be recoverable from the node. Store both key and value in the node.

<br>

#### Q4: "Why does `get` modify the cache? Isn't a read supposed to be const?"

LRU's definition makes a read a *use*, and a use updates recency by moving the node to the front. So `get` is logically non-const — it mutates the ordering. (This is also why a thread-safe LRU can't use a plain read lock for `get` — covered Day 39.) If you want a true non-mutating read, that's a separate `peek()`.

<br>

#### Q5: "Why sentinel nodes in the hand-rolled list?"

Sentinel (dummy) head and tail nodes guarantee every *real* node has non-null `prev` and `next`, so `unlink`/`insertFront` never special-case the ends. This eliminates the most common linked-list bugs (null-deref at the boundaries). The cost is two extra nodes that never hold data.

<br>

#### Q6: "`std::list` version vs hand-rolled — which in real code?"

`std::list`. It's shorter, exception-safe, leak-proof, and equally fast (its nodes are also individually allocated, and splice is O(1)). The hand-rolled version exists to demonstrate pointer mastery in interviews and to allow node-layout tricks (e.g., a custom allocator/arena). Production: `std::list` + `unordered_map`.

<br>

#### Q7: "What's the failure mode of LRU, and what fixes it?"

**Scan pollution**: one large sequential pass (e.g., a full-table scan) touches many keys once, evicting the genuinely hot working set in favor of data that's never reused. Scan-resistant policies — **ARC**, **2Q**, **LRU-K** — protect frequently-used items by requiring more than one access before promotion to the "protected" segment.

<br>

#### Q8: "How would you bound by total *memory* instead of entry count?"

Track a running `m_bytes` sum (each entry contributes its measured size), and evict from the tail while `m_bytes > m_maxBytes`. You need a size function per value type. Many real caches (Caffeine, Guava, Redis `maxmemory`) support a weigher/size-estimator exactly for this.

<br><br>

---

## Reflection Questions

1. Why is neither a hash map alone nor a linked list alone sufficient for O(1) `get` and `put`?
2. What does `splice` do at the pointer level, and why does it preserve iterator validity?
3. Why must a list node carry its key in addition to its value?
4. Why is `get` a *mutating* operation in an LRU cache?
5. What is scan pollution, and which policies mitigate it?
6. When would you bound a cache by bytes instead of entry count, and how?

---

## Interview Questions

1. "Design an LRU cache with O(1) `get` and `put`."
2. "Why do you need both a hash map and a doubly-linked list?"
3. "Explain the splice / move-to-front operation. Why is it O(1)?"
4. "Implement the doubly-linked list by hand, without `std::list`. Why use sentinels?"
5. "Why does `get` change the data structure?"
6. "Compare LRU, LFU, FIFO, and Random eviction. When does each win?"
7. "What is scan pollution and how do ARC/2Q/LRU-K address it?"
8. "How would you add a TTL / expiry to each entry?"
9. "How would you cap the cache by total memory rather than count?"
10. "What goes wrong if you store `std::vector` iterators in the map instead of `std::list` iterators?"

---

**Next**: Day 39 — LRU Cache: Thread-Safe →
