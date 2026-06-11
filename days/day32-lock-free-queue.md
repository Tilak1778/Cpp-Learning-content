# Day 32: Lock-Free Queue

[← Back to Study Plan](../lld-study-plan.md) | [← Day 31](day31-atomics-memory-ordering.md)

> **Time**: ~1.5-2 hours
> **Goal**: A queue is the workhorse of concurrent systems (it's literally the heart of the thread pool). Today we build the *simplest correct* lock-free queue — the **single-producer / single-consumer (SPSC) ring buffer** — and understand *why* it's correct with only relaxed/acquire/release and no CAS. Then survey the hard cases: the **ABA problem**, the **Michael-Scott** multi-producer/multi-consumer queue, and the reclamation techniques (**hazard pointers**, **epoch-based reclamation**) that make MPMC lock-free queues safe. Build the SPSC ring buffer.

---
---

# PART 1: SPSC — THE EASY CASE

---
---

<br>

<h2 style="color: #2980B9;">📘 32.1 Why SPSC Is Special</h2>

The general lock-free queue (any number of producers and consumers) is genuinely hard. But a huge fraction of real systems only need **one producer and one consumer**: a network thread feeding a worker, an audio callback handing samples to a DSP thread, a logging thread draining from a hot path. For exactly *one* writer and *one* reader, the problem collapses to something beautifully simple.

The key insight: with a single producer and a single consumer, **only one thread ever writes each index**. The producer owns the `head` (write index); the consumer owns the `tail` (read index). Neither index is ever the target of a contended write, so **no CAS is needed** — plain atomic loads/stores with acquire/release suffice.

```
Ring buffer (capacity 8):
        consumer reads here        producer writes here
                 │                        │
                 ▼                        ▼
   ┌────┬────┬────┬────┬────┬────┬────┬────┐
   │    │ C  │ D  │ E  │    │    │    │    │
   └────┴────┴────┴────┴────┴────┴────┴────┘
          ▲tail                  ▲head
   (indices wrap modulo capacity)
   empty:  head == tail
   full:   (head + 1) % cap == tail   ← one slot kept empty to disambiguate
```

We keep **one slot always empty** so we can distinguish "full" from "empty" (both would otherwise be `head == tail`). Cost: one wasted slot. Benefit: trivial, branchless emptiness/fullness checks.

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 32.2 The Two Indices and Their Orderings</h2>

`head` = next slot the **producer** will write. `tail` = next slot the **consumer** will read.

- **Producer (`push`)**: reads `tail` (acquire — to see how much space the consumer has freed), checks for full, writes the data into the slot, then advances `head` with a **release** store (publishing the data to the consumer).
- **Consumer (`pop`)**: reads `head` (acquire — to see data the producer published), checks for empty, reads the slot, then advances `tail` with a **release** store (publishing the freed space to the producer).

```cpp
bool push(const T& item) {
    const size_t h = head_.load(std::memory_order_relaxed);   // we own head: relaxed
    const size_t next = increment(h);
    if (next == tail_.load(std::memory_order_acquire))         // full? acquire tail
        return false;
    buffer_[h] = item;                                          // write payload
    head_.store(next, std::memory_order_release);              // publish: data-before-head
    return true;
}

bool pop(T& out) {
    const size_t t = tail_.load(std::memory_order_relaxed);    // we own tail: relaxed
    if (t == head_.load(std::memory_order_acquire))            // empty? acquire head
        return false;
    out = buffer_[t];                                           // read payload
    tail_.store(increment(t), std::memory_order_release);      // publish freed slot
    return true;
}
```

Why these exact orderings:
- The producer's `head_.store(release)` *publishes* `buffer_[h] = item`. The consumer's `head_.load(acquire)` that sees the new head therefore also sees the written payload — release/acquire handshake from Day 31. **Without this, the consumer could read a stale/garbage slot.**
- Symmetrically, the consumer's `tail_.store(release)` publishes the freed slot; the producer's `tail_.load(acquire)` sees it, so it won't overwrite a slot the consumer is still reading.
- Each thread reads *its own* index `relaxed` — it's the only writer, so no synchronization is needed there.

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 32.3 False Sharing — the Hidden Killer</h2>

`head_` and `tail_` are written by *different* threads. If they sit on the **same cache line** (typically 64 bytes), every producer write to `head_` invalidates the consumer's cached copy of the line and vice versa — even though they're logically independent. The cores ping-pong the line back and forth, destroying throughput. This is **false sharing**.

```
Bad layout (same line):           Good layout (separate lines):
┌──────── 64B line ────────┐      ┌─ line A ─┐   ┌─ line B ─┐
│ head_ │ tail_ │  ...     │      │ head_ pad│   │ tail_ pad│
└──────────────────────────┘      └──────────┘   └──────────┘
 ping-pong on EVERY op             producer & consumer
                                   touch different lines
```

Fix: pad each index onto its own cache line.

```cpp
struct alignas(64) PaddedIndex {
    std::atomic<size_t> value{0};
    char pad[64 - sizeof(std::atomic<size_t>)];
};
```

C++17 gives `std::hardware_destructive_interference_size` (the cache-line size to *avoid* sharing) — use it instead of hard-coding 64 where available. False sharing is the most common reason a "correct" lock-free queue is slower than a mutex-based one; always check the layout.

<br><br>

---
---

# PART 2: THE HARD CASES (OVERVIEW)

---
---

<br>

<h2 style="color: #2980B9;">📘 32.4 The ABA Problem (Revisited)</h2>

Day 31 introduced ABA on the Treiber stack; it's even more central to MPMC queues. Recap with a sharp example:

```
Thread 1: reads head = A, intends CAS(head, A, A->next)
          ... preempted ...
Thread 2: pop A; pop B; A is freed; allocator REUSES A's address for a new node;
          push that node → head = A again (same pointer value, different node!)
Thread 1: resumes. CAS(head, A, ...) sees head == A → SUCCEEDS,
          but the structure underneath changed completely. Corruption.
```

The CAS only compares the **pointer value**, not the *history*. A→B→A is invisible to it. Multi-producer/consumer queues hit this constantly because freed nodes get recycled.

Defenses:

| Technique | How it defeats ABA |
|-----------|--------------------|
| **Tagged pointers** | Pack a monotonically-increasing version counter beside the pointer; CAS compares pointer+tag together (needs DWCAS / 128-bit atomic). A→B→A bumps the tag, so the CAS fails. |
| **Hazard pointers** | Don't reuse a node's memory while any thread might still reference it — so the address can't come back as "A" during the window. |
| **Epoch / RCU** | Defer reclamation to an epoch when no thread holds an old reference. Same effect: no premature reuse. |

SPSC sidesteps ABA entirely — there's no CAS and no node reuse to confuse.

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 32.5 The Michael-Scott Queue (MPMC)</h2>

The canonical lock-free FIFO queue (Michael & Scott, 1996) is a singly-linked list with separate atomic `head` and `tail` pointers and a **dummy/sentinel node** that decouples them so producers and consumers rarely touch the same node.

```
   head ──► [dummy] ──► [v1] ──► [v2] ──► null ◄── tail
            (consumed) (data)   (data)
   enqueue: CAS tail->next from null to new node, then swing tail
   dequeue: read head->next's value, CAS head forward
```

Two clever ideas make it work:

1. **Sentinel node** — the queue is never empty of *nodes* (always at least the dummy), so `head` and `tail` are never null and the empty/non-empty logic has no special cases.
2. **Helping / two-step enqueue** — `tail` can lag one node behind (after a producer links a node but before it swings `tail`). Any thread that notices `tail->next != null` *helps* advance `tail` with a CAS before proceeding. This makes the algorithm **lock-free** (system-wide progress guaranteed) even if a producer is preempted mid-enqueue.

The enqueue/dequeue each use CAS loops, so the queue inherits the **ABA problem** and the **reclamation problem** — you cannot just `delete` a dequeued node, because another thread may still be reading it. That's what §32.6 addresses. The MS queue is what underlies `boost::lockfree::queue` and many production MPMC queues; building one correctly is a multi-day exercise, which is why today's *build* is the SPSC version.

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 32.6 Safe Memory Reclamation</h2>

The deepest problem in lock-free data structures isn't the algorithm — it's **when is it safe to free a node?** A node removed from the structure might still be pointed at by another thread that read the pointer a moment ago. `delete` it and you get a use-after-free; never free it and you leak. Three industrial answers:

| Scheme | Mechanism | Trade-off |
|--------|-----------|-----------|
| **Reference counting** | Atomic refcount per node; free at zero | Simple, but the count itself is a contended atomic and has its own ABA subtleties |
| **Hazard pointers** | Each thread publishes the pointers it's *currently* dereferencing into a global per-thread "hazard" slot. A node is freed only when no hazard pointer references it. | Bounded memory, but a scan of all hazard slots per reclamation; reader overhead per access |
| **Epoch-based reclamation (EBR/RCU)** | Threads enter/exit "epochs." A retired node is freed only after all threads have advanced past the epoch in which it was retired. | Very low reader overhead, but a single stalled thread can stall reclamation (unbounded memory) |

```
Hazard pointer flow:
  reader:  hazard[me] = node;  re-verify node still in structure;  use node;  hazard[me] = null
  retirer: unlink node;  add to retire-list;  periodically: free nodes
           NOT present in ANY thread's hazard slot.
```

For an interview, the expected answer to "how do you safely free in a lock-free queue?" is: **"You can't just `delete` it — a concurrent reader may hold the pointer. Use hazard pointers or epoch-based reclamation (RCU) to defer the free until no one references it."** C++26 is standardizing `std::hazard_pointer`; until then, libraries (folly, libcds) provide them.

<br><br>

---
---

# PART 3: BUILD EXERCISE

---
---

<br>

<h2 style="color: #2980B9;">📘 32.7 Exercise: SPSC Ring Buffer</h2>

Build a fixed-capacity, single-producer/single-consumer lock-free ring buffer. No CAS, no reclamation problem — just correct acquire/release and a false-sharing-free layout. Verify with a streaming stress test that proves no items are lost, duplicated, or reordered.

<br>

#### Skeleton

```cpp
// spsc_ring.h
#pragma once
#include <atomic>
#include <cstddef>
#include <new>          // std::hardware_destructive_interference_size
#include <vector>
#include <utility>

#if defined(__cpp_lib_hardware_interference_size)
  inline constexpr std::size_t kCacheLine =
      std::hardware_destructive_interference_size;
#else
  inline constexpr std::size_t kCacheLine = 64;
#endif

template <typename T>
class SpscRingBuffer {
public:
    // capacity must be >= 2; one slot is reserved to distinguish full/empty.
    explicit SpscRingBuffer(std::size_t capacity)
        : m_cap(capacity), m_buf(capacity) {}

    SpscRingBuffer(const SpscRingBuffer&)            = delete;
    SpscRingBuffer& operator=(const SpscRingBuffer&) = delete;

    // Called by the PRODUCER thread only.
    bool push(const T& item) {
        const std::size_t h    = m_head.load(std::memory_order_relaxed);
        const std::size_t next = inc(h);
        if (next == m_tail.load(std::memory_order_acquire))
            return false;                                   // full
        m_buf[h] = item;
        m_head.store(next, std::memory_order_release);      // publish payload
        return true;
    }

    // Called by the CONSUMER thread only.
    bool pop(T& out) {
        const std::size_t t = m_tail.load(std::memory_order_relaxed);
        if (t == m_head.load(std::memory_order_acquire))
            return false;                                   // empty
        out = std::move(m_buf[t]);
        m_tail.store(inc(t), std::memory_order_release);    // publish freed slot
        return true;
    }

    bool empty() const {
        return m_head.load(std::memory_order_acquire) ==
               m_tail.load(std::memory_order_acquire);
    }

private:
    std::size_t inc(std::size_t i) const { return (i + 1) % m_cap; }

    const std::size_t m_cap;
    std::vector<T>    m_buf;

    // Pad head and tail onto separate cache lines to kill false sharing.
    alignas(kCacheLine) std::atomic<std::size_t> m_head{0};   // producer writes
    alignas(kCacheLine) std::atomic<std::size_t> m_tail{0};   // consumer writes
};
```

<br>

#### Test driver

```cpp
// main.cpp
#include "spsc_ring.h"
#include <atomic>
#include <cassert>
#include <iostream>
#include <thread>

int main() {
    std::cout << "=== 1. Single-threaded FIFO + full/empty ===\n";
    {
        SpscRingBuffer<int> q(4);     // usable capacity = 3 (one reserved)
        int v;
        assert(q.empty());
        assert(!q.pop(v));            // empty
        assert(q.push(10));
        assert(q.push(20));
        assert(q.push(30));
        assert(!q.push(40));          // full (one slot reserved)
        assert(q.pop(v) && v == 10);  // FIFO
        assert(q.pop(v) && v == 20);
        assert(q.push(40));           // room again
        assert(q.pop(v) && v == 30);
        assert(q.pop(v) && v == 40);
        assert(!q.pop(v));
        std::cout << "  FIFO + full/empty verified.\n";
    }

    std::cout << "=== 2. Concurrent streaming: nothing lost, order preserved ===\n";
    {
        constexpr long N = 5'000'000;
        SpscRingBuffer<long> q(1024);

        std::thread producer([&] {
            for (long i = 0; i < N; ++i)
                while (!q.push(i)) { /* spin until space */ }
        });

        long received = 0, expected = 0;
        bool orderOk = true;
        std::thread consumer([&] {
            long v;
            while (received < N) {
                if (q.pop(v)) {
                    if (v != expected) orderOk = false;  // SPSC must be in-order
                    ++expected;
                    ++received;
                }
            }
        });

        producer.join();
        consumer.join();
        std::cout << "  received=" << received
                  << " in_order=" << (orderOk ? "yes" : "NO") << "\n";
        assert(received == N);
        assert(orderOk);
    }

    std::cout << "All assertions passed.\n";
    return 0;
}
```

<br>

#### Compile and run

```bash
g++ -std=c++17 -Wall -Wextra -Wpedantic -fsanitize=thread -pthread \
    -O1 -o day32 main.cpp && ./day32
```

(`-O1` keeps TSan's instrumentation manageable on 5M iterations; drop the count if it's slow. ThreadSanitizer should report **no** data races — the acquire/release pairing is what guarantees that.)

<br>

#### Expected output

```
=== 1. Single-threaded FIFO + full/empty ===
  FIFO + full/empty verified.
=== 2. Concurrent streaming: nothing lost, order preserved ===
  received=5000000 in_order=yes
All assertions passed.
```

<br>

#### Bonus Challenges

1. **Demonstrate false sharing** — remove the `alignas(kCacheLine)` so `m_head` and `m_tail` share a line. Benchmark throughput (items/sec) with and without padding. Expect a large slowdown without it.

2. **Power-of-two capacity** — require `m_cap` to be a power of two and replace `% m_cap` with `& (m_cap - 1)`. Modulo on a non-power-of-two is a division; the mask is one instruction. Measure the difference.

3. **Batch operations** — add `push_n` / `pop_n` that move multiple items per index update, amortizing the atomic store. This is a big throughput win for streaming workloads.

4. **Cache the other index** — have the producer keep a *local* copy of the last-seen `tail` and only reload it (acquire) when the cached value says "full." This avoids an atomic load on the common path. (folly's `ProducerConsumerQueue` does this.)

5. **Move-only / non-trivial T** — make `push` take `T&&` and use placement-new + explicit destructor management so the buffer works for `std::unique_ptr<T>` and other move-only types without default-constructing every slot.

6. **Blocking variant** — add `condition_variable`-backed `push_blocking`/`pop_blocking` that sleep instead of spin when full/empty, while keeping the lock-free fast path when there's space/data.

<br><br>

---
---

# PART 4: DEEP DIVE — COMMON QUESTIONS

---
---

<br>

<h2 style="color: #2980B9;">📘 32.8 Q&A</h2>

<br>

#### Q1: "Why doesn't an SPSC ring buffer need CAS?"

Because each index has exactly **one** writer. The producer is the sole writer of `head`; the consumer is the sole writer of `tail`. There's never a write-write contention to resolve, so there's nothing for compare-and-swap to guard against. Plain atomic load/store with acquire/release is sufficient and far cheaper than a CAS loop. CAS is only needed when *multiple* threads write the *same* location.

<br>

#### Q2: "Why reserve one empty slot?"

To distinguish *full* from *empty*. Both would otherwise be `head == tail`. By treating "the slot just before tail" as the boundary (`(head+1)%cap == tail` means full), full and empty become unambiguous at the cost of one unused slot. Alternatives: a separate atomic count, or storing absolute (non-wrapped) indices and comparing `head - tail`.

<br>

#### Q3: "Which memory orders and why?"

Each thread reads its *own* index `relaxed` (sole writer, no sync needed). The producer's `head.store` is `release` and the consumer's `head.load` is `acquire` — this handshake publishes the written payload before the index advance is observed. Symmetrically for `tail` (publishing freed space). The release ensures the data write can't be reordered after the index update; the acquire ensures the reader sees the data once it sees the new index.

<br>

#### Q4: "What is false sharing and how does it hurt here?"

False sharing is when two independent variables share a cache line, so writing one invalidates the other in remote caches even though there's no logical dependency. `head` (written by the producer) and `tail` (written by the consumer) on the same line cause a cache-line ping-pong on every operation. Padding each onto its own line (`alignas(64)` / `hardware_destructive_interference_size`) eliminates it and can multiply throughput.

<br>

#### Q5: "What is the ABA problem and does SPSC have it?"

ABA: a CAS reads value A, the value changes to B and back to A (often via memory reuse), and the CAS can't tell — it succeeds wrongly. SPSC has **no** ABA because it uses no CAS and recycles slots by index, not by reusing freed pointers. ABA bites pointer-based MPMC structures (Treiber stack, Michael-Scott queue) where freed nodes are reallocated.

<br>

#### Q6: "How does the Michael-Scott queue stay lock-free if a producer is preempted mid-enqueue?"

Via **helping**. Enqueue is two steps: CAS the tail node's `next` to the new node, then swing `tail`. If a producer completes step 1 and is preempted before step 2, `tail` lags. Any other thread that observes `tail->next != null` performs the `tail`-swing CAS *on the stalled producer's behalf* before doing its own work. So the system always makes progress regardless of any single thread's scheduling — that's the lock-free guarantee.

<br>

#### Q7: "Why can't I just `delete` a dequeued node in a lock-free MPMC queue?"

Because another thread may have read the pointer to that node a moment before you removed it and is about to dereference it. `delete` creates a use-after-free, and worse, the freed address may be reallocated and trigger ABA. You must *defer* reclamation until no thread can reference the node — via hazard pointers (publish in-use pointers, free only unreferenced ones) or epoch-based reclamation (free only after all threads pass the retirement epoch).

<br>

#### Q8: "Lock-free vs wait-free vs obstruction-free — what's the difference?"

- **Obstruction-free**: a thread makes progress if it runs *in isolation* (others paused). Weakest.
- **Lock-free**: the *system* always makes progress — *some* thread completes in a bounded number of steps, even if individuals retry forever. (Michael-Scott queue, Treiber stack.)
- **Wait-free**: *every* thread completes in a bounded number of steps — no starvation, no unbounded retries. Strongest and hardest. (SPSC push/pop are effectively wait-free per operation: no loops.)

<br><br>

---

## Reflection Questions

1. Why does single-producer/single-consumer let you avoid CAS entirely?
2. Walk the acquire/release pairing on `head` and on `tail`. What would break if you used `relaxed` everywhere?
3. What is false sharing, why do `head` and `tail` trigger it, and how do you fix it?
4. Explain the ABA problem with a concrete pointer-reuse sequence. Why is SPSC immune?
5. What two ideas (sentinel node, helping) make the Michael-Scott queue work and stay lock-free?
6. Why can't you `delete` a dequeued node in an MPMC lock-free queue, and what are the two main fixes?

---

## Interview Questions

1. "Implement a single-producer/single-consumer lock-free ring buffer. Justify every memory order."
2. "Why does SPSC not need compare-and-swap, but MPMC does?"
3. "What is false sharing? Show how it can make a lock-free queue slower than a mutex."
4. "Explain the ABA problem and three ways to defend against it."
5. "Describe the Michael-Scott queue. What roles do the sentinel node and 'helping' play?"
6. "How do you safely reclaim memory in a lock-free queue? Compare hazard pointers and epoch-based reclamation."
7. "Define lock-free, wait-free, and obstruction-free, with an example of each."
8. "Why reserve an empty slot in a ring buffer, and what are the alternatives?"
9. "How would you make the SPSC buffer faster: index caching, batching, power-of-two masking — explain each."
10. "When would you choose a mutex-protected `std::queue` over a lock-free queue?"

---

**Next**: Day 33 — Actor Model →
