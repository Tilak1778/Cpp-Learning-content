# Day 31: Atomics & Memory Ordering

[← Back to Study Plan](../lld-study-plan.md) | [← Day 30](day30-thread-pool-refinement.md)

> **Time**: ~1.5-2 hours
> **Goal**: This is the conceptual core of lock-free programming. Learn what `std::atomic` actually guarantees (atomicity vs ordering — two separate things), the six `memory_order` values and what each permits the compiler/CPU to reorder, the **happens-before** relation that makes a release/acquire pair work, why **CAS loops** are the universal lock-free building block, and the `compare_exchange_weak` vs `strong` distinction (spurious failure). Build a lock-free **Treiber stack** with `compare_exchange_weak`.

---
---

# PART 1: ATOMICITY IS NOT ORDERING

---
---

<br>

<h2 style="color: #2980B9;">📘 31.1 Two Separate Guarantees</h2>

`std::atomic<T>` gives you **two** distinct things, and conflating them is the #1 source of confusion:

1. **Atomicity** — a load or store happens all-at-once; no thread ever sees a half-written value (no *torn* reads/writes). This alone is what people usually mean by "atomic."
2. **Ordering** — constraints on how operations *around* the atomic can be reordered by the compiler and CPU, and on when one thread's writes become *visible* to another.

A plain `int counter++` from two threads is a data race (undefined behaviour) because the read-modify-write isn't atomic. `std::atomic<int>` fixes the atomicity. But the *ordering* of surrounding non-atomic memory is a separate dial — the `memory_order` argument.

```
Thread A                         Thread B
  data = 42;          // (1)       if (ready.load()) {   // (3)
  ready.store(true);  // (2)           use(data);        // (4)
                                   }
```

For B to safely `use(data)` after seeing `ready == true`, the store (2) must *synchronize-with* the load (3), and (1) must be guaranteed visible before (4). Atomicity of `ready` alone does **not** guarantee this — you also need the right *ordering*. That's what acquire/release provide.

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 31.2 Why Reordering Exists</h2>

Both the **compiler** and the **CPU** reorder memory operations for speed, as long as a *single thread* can't tell:

- **Compiler**: reorders instructions, keeps values in registers, eliminates "redundant" loads.
- **CPU**: store buffers delay writes; out-of-order execution and speculation run loads early; caches make a write visible to other cores at different times.

In single-threaded code this is invisible (the "as-if" rule). In multithreaded code, another thread *can* observe the reordering. Memory ordering is how you tell the compiler and CPU: "these particular reorderings would be observable and must not happen."

Without atomics/fences, the compiler might hoist `ready` into a register and B loops forever, or A's `data = 42` might land in B's view *after* `ready = true`. Both are legal for non-synchronized code.

<br><br>

---
---

# PART 2: THE MEMORY ORDERS

---
---

<br>

<h2 style="color: #2980B9;">📘 31.3 The Six Values</h2>

```cpp
enum class memory_order {
    relaxed, consume, acquire, release, acq_rel, seq_cst
};
```

| Order | Applies to | Guarantee |
|-------|-----------|-----------|
| `relaxed` | load/store/RMW | Atomicity only. **No** ordering w.r.t. other memory. |
| `acquire` | load (and RMW) | No reads/writes *after* it can move *before* it. Sees writes released by the matching store. |
| `release` | store (and RMW) | No reads/writes *before* it can move *after* it. Publishes prior writes to the acquiring load. |
| `acq_rel` | RMW only | Both acquire (on its load half) and release (on its store half). |
| `seq_cst` | load/store/RMW | Acquire+release **plus** a single global total order over all seq_cst ops. The default; the strongest and slowest. |
| `consume` | load | A weaker acquire limited to data-dependent reads. **Effectively deprecated** — every compiler promotes it to `acquire`. Don't use it. |

The mnemonic: **acquire is a one-way barrier downward** (things can't float up past it), **release is a one-way barrier upward** (things can't sink down past it). Together they fence a critical section.

```
        ─── acquire load ───        nothing below can move ABOVE this line
        │  reads/writes here  │  ←  this region is "after acquire, before release"
        ─── release store ──        nothing above can move BELOW this line
```

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 31.4 The Release/Acquire Handshake</h2>

The fundamental synchronization tool. A `release` store and an `acquire` load on the **same atomic** form a *synchronizes-with* edge when the acquire reads the value the release wrote:

```cpp
std::atomic<bool> ready{false};
int data = 0;                       // plain (non-atomic) data

// Producer
data = 42;                          // (A)  ordinary write
ready.store(true, std::memory_order_release);   // (B)  RELEASE

// Consumer
while (!ready.load(std::memory_order_acquire))  // (C)  ACQUIRE
    ;                                            // spin
assert(data == 42);                 // (D)  GUARANTEED to see 42
```

Why (D) is guaranteed:

```
  (A) data = 42
       │ sequenced-before (same thread, program order)
       ▼
  (B) ready.store(release) ─────synchronizes-with────► (C) ready.load(acquire)
                                                              │ sequenced-before
                                                              ▼
                                                       (D) read data == 42

  Transitively:  (A) happens-before (D)   ⇒  (D) must observe (A)'s write.
```

The release store (B) acts as a **publish**: every write *before* it in program order (including the non-atomic `data = 42`) becomes visible to any thread whose acquire load (C) reads the published value. This is how you safely transfer plain data between threads using just one atomic flag — no mutex.

<br>

#### Happens-before, precisely

*Happens-before* is the partial order that decides what a thread is guaranteed to see:

- **Sequenced-before**: within one thread, program order.
- **Synchronizes-with**: a release op synchronizes with an acquire op that reads its value (on the same atomic).
- **Happens-before** = transitive closure of the two above.

If write X *happens-before* read Y, then Y sees X (or a later write). If neither happens-before the other and at least one is a write → **data race → undefined behaviour**.

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 31.5 Relaxed: Atomic but Unordered</h2>

`relaxed` guarantees atomicity and *nothing else* — no synchronizes-with, no happens-before with other variables. Its classic correct use is a **counter** where you only care about the final total, not ordering:

```cpp
std::atomic<int> hits{0};
// many threads:
hits.fetch_add(1, std::memory_order_relaxed);   // count is exact; order irrelevant
// ... later, after a barrier/join ...
std::cout << hits.load(std::memory_order_relaxed);
```

Each `fetch_add` is atomic, so no increments are lost. But you must *not* use a relaxed flag to publish other data — there's no ordering, so a reader seeing the flag set has no guarantee about prior writes. Relaxed is for "statistics," reference counts (with care — see Day 4's atomic refcount, where the *decrement-to-zero* needs an acquire fence), and rarely as the building block under explicit fences.

<br>

#### seq_cst: the safe default

`seq_cst` is acquire+release plus a single global total order that all threads agree on. It's the default for `atomic<T>` operations (`x.store(v)` with no order argument is `seq_cst`). It rules out the surprising **Independent-Reads-of-Independent-Writes** anomalies that acquire/release alone permit. Start with `seq_cst`; weaken to acquire/release/relaxed *only* when profiling shows it matters and you've proven the weaker ordering is correct.

<br><br>

---
---

# PART 3: COMPARE-AND-SWAP (CAS)

---
---

<br>

<h2 style="color: #2980B9;">📘 31.6 The Universal Primitive</h2>

`fetch_add` is great for counters, but most lock-free structures need to update a pointer conditionally: "set `head` to `newNode`, but only if `head` is still what I last read." That's **compare-and-swap**:

```cpp
bool compare_exchange_strong(T& expected, T desired, memory_order);
// Atomically:
//   if (*this == expected) { *this = desired; return true;  }
//   else                   { expected = *this; return false; }   // expected updated!
```

The crucial detail: on **failure**, `expected` is **overwritten** with the current value. This is what makes the CAS *loop* idiomatic — you don't re-read manually; the failed CAS hands you the fresh value:

```cpp
T old = value.load(std::memory_order_relaxed);
T desired;
do {
    desired = compute_new_value(old);     // recompute from the latest 'old'
} while (!value.compare_exchange_weak(old, desired,
                                      std::memory_order_release,
                                      std::memory_order_relaxed));
//        on failure, 'old' is refreshed to the current value → retry
```

This loop is the heartbeat of lock-free code: read, compute a new value, try to swap it in atomically; if someone else changed it meanwhile, retry with the updated value. It's optimistic concurrency — no locks, just "try and retry."

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 31.7 weak vs strong</h2>

There are two flavours:

| | `compare_exchange_strong` | `compare_exchange_weak` |
|---|---------------------------|--------------------------|
| **Spurious failure** | Never | Allowed — may return false even when `*this == expected` |
| **Codegen** | May need an internal loop on LL/SC architectures (ARM, POWER) | Maps directly to one LL/SC attempt — cheaper |
| **Use when** | Not already looping (single attempt, e.g. an `if`) | Already inside a retry loop |

**Spurious failure** comes from *Load-Linked / Store-Conditional* (LL/SC) hardware (ARM, RISC-V, POWER): the store-conditional can fail for reasons unrelated to the value — a cache-line eviction, an interrupt, a context switch between the LL and SC. `compare_exchange_weak` exposes that possibility; `strong` hides it by retrying internally.

The rule:
- **Inside a loop** → use `weak`. A spurious failure just costs one extra harmless iteration, and you avoid the hidden inner loop `strong` would add. This is the common case in lock-free data structures.
- **Not in a loop** (you want a single decisive attempt) → use `strong`, so a spurious failure doesn't make you wrongly conclude the value changed.

On x86 (which has a native CAS, `cmpxchg`) there is no spurious failure and the two compile identically — but write portable code as if weak can fail spuriously, because it can on ARM.

<br>

#### The two memory orders of CAS

`compare_exchange_*` takes **two** orders: one for **success** (the RMW that publishes — usually `release` or `acq_rel`) and one for **failure** (just a load — `relaxed` or `acquire`). The failure order may not be *stronger* than the success order, and may not be `release`/`acq_rel` (a failed CAS does no store, so it can't release).

<br><br>

---
---

# PART 4: THE TREIBER STACK

---
---

<br>

<h2 style="color: #2980B9;">📘 31.8 A Lock-Free Stack with One CAS</h2>

The Treiber stack (R. Kent Treiber, 1986) is the "hello world" of lock-free structures: a singly-linked LIFO stack whose `push` and `pop` are each a single CAS loop on the head pointer.

```
push(x):  new node N → N.next = head;  CAS(head, head, N)
                       ┌───┐    ┌───┐    ┌───┐
            head ─────►│ N │───►│ A │───►│ B │───► null
                       └───┘    └───┘    └───┘

pop():    read head H; CAS(head, H, H->next)  → return H's value
```

```cpp
template <typename T>
class TreiberStack {
    struct Node {
        T data;
        Node* next;
        explicit Node(T d) : data(std::move(d)), next(nullptr) {}
    };
    std::atomic<Node*> m_head{nullptr};

public:
    void push(T value) {
        Node* node = new Node(std::move(value));
        node->next = m_head.load(std::memory_order_relaxed);
        // Publish 'node' (and its initialized fields) via release on success.
        while (!m_head.compare_exchange_weak(
                   node->next, node,
                   std::memory_order_release,   // success: publish the new node
                   std::memory_order_relaxed))  // failure: just reloads node->next
        {
            // On failure, compare_exchange updates node->next to the current
            // head for us — so the retry links correctly. No manual reload.
        }
    }

    bool pop(T& out) {
        Node* old = m_head.load(std::memory_order_acquire);
        while (old &&
               !m_head.compare_exchange_weak(
                   old, old->next,
                   std::memory_order_acquire,   // success: acquire the node we popped
                   std::memory_order_acquire))  // failure: re-acquire current head
        {
            // 'old' refreshed to current head on failure; loop retries.
        }
        if (!old) return false;
        out = std::move(old->data);
        delete old;                              // see §31.9 — this is the unsafe part!
        return true;
    }

    ~TreiberStack() {
        Node* p = m_head.load(std::memory_order_relaxed);
        while (p) { Node* n = p->next; delete p; p = n; }
    }
};
```

Why these orderings:
- **push** uses `release` on success so the fully-initialized `Node` (its `data` and `next`) is published to any `pop` that later acquires it. Failure order is `relaxed` — a failed CAS does no publishing, and `node->next` is just being refreshed.
- **pop** uses `acquire` so reading `old->next` and `old->data` sees the writes the pushing thread released. We need `old->next` *before* the CAS commits, so the load that feeds the CAS must synchronize with the push.

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 31.9 What This Stack Does NOT Solve</h2>

The Treiber stack above is correct for the *pointer juggling* but has **two** well-known lock-free hazards that a real implementation must address — and which we deliberately leave for Day 32:

#### The ABA problem

Thread 1 reads `head = A` (with `A->next = B`), then stalls. Thread 2 pops A, pops B, and pushes A again (often A was freed and a *new* node reuses A's address). Now `head = A` again. Thread 1 resumes, its CAS sees `head == A` and **succeeds** — but sets `head = B`, which has since been freed. The value changed A→B→A, and the CAS couldn't tell. Classic ABA.

```
T1: read head=A, plan head:=A->next (=B), then PAUSE
T2: pop A, pop B, free both, push A again   (head=A, but A->next is now garbage)
T1: CAS(head, A, B) succeeds  ← WRONG: B was freed
```

#### Safe reclamation (the `delete` problem)

`pop` calls `delete old` — but another thread might still hold a pointer to `old` from a concurrent `pop` that read the head before we removed it. Freeing it is a use-after-free. Real lock-free stacks defer reclamation with **hazard pointers** or **epoch-based reclamation** (RCU-style), or simply leak / use a free-list. These are Day 32 topics.

Under ThreadSanitizer with high contention you may see the use-after-free; the structure is a teaching tool. The pointer-CAS logic is the lesson today.

<br><br>

---
---

# PART 5: BUILD EXERCISE

---
---

<br>

<h2 style="color: #2980B9;">📘 31.10 Exercise: Treiber Stack + Memory-Order Lab</h2>

Implement the Treiber stack and verify it under contention; also run the release/acquire handshake to internalize happens-before.

<br>

#### Skeleton

```cpp
// treiber_stack.h
#pragma once
#include <atomic>
#include <utility>

template <typename T>
class TreiberStack {
    struct Node {
        T     data;
        Node* next;
        explicit Node(T d) : data(std::move(d)), next(nullptr) {}
    };
    std::atomic<Node*> m_head{nullptr};

public:
    TreiberStack() = default;
    TreiberStack(const TreiberStack&)            = delete;
    TreiberStack& operator=(const TreiberStack&) = delete;

    void push(T value) {
        Node* node = new Node(std::move(value));
        node->next = m_head.load(std::memory_order_relaxed);
        while (!m_head.compare_exchange_weak(node->next, node,
                                             std::memory_order_release,
                                             std::memory_order_relaxed)) {
        }
    }

    bool pop(T& out) {
        Node* old = m_head.load(std::memory_order_acquire);
        while (old && !m_head.compare_exchange_weak(old, old->next,
                                                    std::memory_order_acquire,
                                                    std::memory_order_acquire)) {
        }
        if (!old) return false;
        out = std::move(old->data);
        delete old;        // NB: unsafe reclamation — see Day 32
        return true;
    }

    bool empty() const { return m_head.load(std::memory_order_acquire) == nullptr; }

    ~TreiberStack() {
        Node* p = m_head.load(std::memory_order_relaxed);
        while (p) { Node* n = p->next; delete p; p = n; }
    }
};
```

<br>

#### Test driver

```cpp
// main.cpp
#include "treiber_stack.h"
#include <atomic>
#include <cassert>
#include <iostream>
#include <thread>
#include <vector>

// --- Part A: single-threaded correctness ---
void testSingleThreaded() {
    std::cout << "=== Single-threaded LIFO ===\n";
    TreiberStack<int> s;
    assert(s.empty());
    s.push(1); s.push(2); s.push(3);
    int v;
    assert(s.pop(v) && v == 3);
    assert(s.pop(v) && v == 2);
    assert(s.pop(v) && v == 1);
    assert(!s.pop(v));
    std::cout << "  LIFO order verified.\n";
}

// --- Part B: concurrent push/pop, conservation of items ---
void testConcurrent() {
    std::cout << "=== Concurrent stress (push==pop count) ===\n";
    TreiberStack<int> s;
    constexpr int kProducers = 4, kPerProducer = 20000;
    std::atomic<int> popped{0};
    std::atomic<bool> doneProducing{false};

    std::vector<std::thread> producers, consumers;
    for (int p = 0; p < kProducers; ++p)
        producers.emplace_back([&s, p] {
            for (int i = 0; i < kPerProducer; ++i)
                s.push(p * kPerProducer + i);
        });

    for (int c = 0; c < 4; ++c)
        consumers.emplace_back([&] {
            int v;
            while (!doneProducing.load(std::memory_order_acquire) || !s.empty()) {
                if (s.pop(v)) popped.fetch_add(1, std::memory_order_relaxed);
            }
        });

    for (auto& t : producers) t.join();
    doneProducing.store(true, std::memory_order_release);
    for (auto& t : consumers) t.join();

    std::cout << "  pushed=" << kProducers * kPerProducer
              << " popped=" << popped.load() << "\n";
    assert(popped.load() == kProducers * kPerProducer);
}

// --- Part C: the release/acquire handshake ---
void testHandshake() {
    std::cout << "=== Release/acquire handshake ===\n";
    std::atomic<bool> ready{false};
    int data = 0;
    std::thread producer([&] {
        data = 42;                                       // (A)
        ready.store(true, std::memory_order_release);    // (B) publish
    });
    std::thread consumer([&] {
        while (!ready.load(std::memory_order_acquire))   // (C) acquire
            std::this_thread::yield();
        assert(data == 42);                              // (D) guaranteed
        std::cout << "  consumer saw data == " << data << "\n";
    });
    producer.join();
    consumer.join();
}

int main() {
    testSingleThreaded();
    testConcurrent();
    testHandshake();
    std::cout << "All assertions passed.\n";
    return 0;
}
```

<br>

#### Compile and run

```bash
g++ -std=c++17 -Wall -Wextra -Wpedantic -fsanitize=thread -pthread \
    -o day31 main.cpp && ./day31
```

> ThreadSanitizer may flag the `delete old` as a use-after-free under heavy contention — that's the **expected** Day 32 reclamation problem, not a bug in the CAS logic. To run clean for now, comment out the `delete` (intentionally leaking) and re-run; the conservation assertion should still hold.

<br>

#### Expected output

```
=== Single-threaded LIFO ===
  LIFO order verified.
=== Concurrent stress (push==pop count) ===
  pushed=80000 popped=80000
=== Release/acquire handshake ===
  consumer saw data == 42
All assertions passed.
```

<br>

#### Bonus Challenges

1. **Break the handshake on purpose** — change (B) and (C) to `memory_order_relaxed`. The assertion *may* still pass on x86 (strong memory model) but is UB and can fail on ARM. Discuss why x86 hides the bug and ARM exposes it.

2. **Counter shootout** — increment a shared counter 10M times from 8 threads using (a) `relaxed`, (b) `seq_cst`, (c) a `std::mutex`. Time all three. Explain the ordering of results.

3. **strong vs weak** — rewrite `push` with `compare_exchange_strong` and benchmark on ARM (or QEMU-emulated AArch64) vs x86. On x86 they should be identical; on ARM, weak-in-a-loop should win.

4. **Size counter** — add an atomic `size_` updated alongside push/pop. Decide its memory order and justify it. (Hint: it's a statistic — `relaxed` is fine.)

5. **Fix ABA with a tagged pointer** — pack a version counter into the pointer (or use `atomic<pair>` / 128-bit DWCAS via `__int128`) so A→B→A is detected. Verify the CAS now fails when the tag changed.

6. **Fences instead of per-op orders** — rewrite using `std::atomic_thread_fence(memory_order_release/acquire)` with relaxed atomics, and explain how the standalone fence relates to a release/acquire on the operation itself.

<br><br>

---
---

# PART 6: DEEP DIVE — COMMON QUESTIONS

---
---

<br>

<h2 style="color: #2980B9;">📘 31.11 Q&A</h2>

<br>

#### Q1: "What's the difference between atomicity and ordering?"

Atomicity means an operation is indivisible — no torn reads/writes. Ordering means constraints on how surrounding memory operations are reordered and when writes become visible to other threads. `std::atomic` always gives atomicity; the `memory_order` argument controls ordering. You can have atomicity without ordering (`relaxed`) but not the reverse.

<br>

#### Q2: "Explain happens-before in one paragraph."

Happens-before is the partial order that determines what a thread is guaranteed to observe. It's built from *sequenced-before* (program order within a thread) and *synchronizes-with* (a release store synchronizes with an acquire load that reads its value), closed transitively. If write X happens-before read Y, Y sees X. If two conflicting accesses (≥1 write) have *no* happens-before relation, you have a data race → undefined behaviour.

<br>

#### Q3: "Why does a release store 'publish' non-atomic data?"

Because *everything sequenced-before* the release store in program order is, transitively, happens-before any acquire that reads the released value. So writing `data = 42; flag.store(true, release);` makes `data = 42` visible to a thread that does `flag.load(acquire)` and sees `true`. The single atomic flag carries all prior writes with it — that's the whole point of release/acquire.

<br>

#### Q4: "When is `relaxed` actually safe?"

When you need atomicity but not ordering relative to other variables: pure counters/statistics where only the final total matters, or as a building block paired with explicit fences. It is *not* safe to use a relaxed flag to publish other data — there's no synchronizes-with, so a reader has no guarantee about writes that preceded the flag.

<br>

#### Q5: "weak vs strong CAS — which do I use?"

Inside a retry loop, use `weak`: it maps to a single LL/SC attempt and a spurious failure just costs one extra loop iteration. Use `strong` when you make a *single* decisive attempt (not looping), so a spurious failure doesn't make you wrongly conclude the value changed. On x86 they're identical (native `cmpxchg`); the distinction matters on LL/SC architectures like ARM/POWER/RISC-V.

<br>

#### Q6: "Why does a failed `compare_exchange` modify `expected`?"

So your retry loop has the fresh value automatically. On failure, `expected` is set to the current contents of the atomic. The idiomatic loop `do { desired = f(old); } while (!cas(old, desired));` relies on this — you never manually re-read; the failed CAS refreshes `old` for you.

<br>

#### Q7: "What are the two memory orders on `compare_exchange`?"

One for **success** (the RMW completes a store, so it can be `release`/`acq_rel`/`seq_cst` to publish) and one for **failure** (no store happens, so only a load order — `relaxed`/`acquire`/`seq_cst`; never `release`/`acq_rel`). The failure order must not be stronger than the success order. In the Treiber `push`, success is `release` (publish the node) and failure is `relaxed` (just refreshing `node->next`).

<br>

#### Q8: "If my code works on x86 with relaxed everywhere, isn't it fine?"

No — it's still undefined behaviour, and x86's strong memory model merely *hides* the bug. x86 doesn't reorder stores past loads of other addresses the way ARM/POWER do, so many incorrect orderings happen to work there. The same binary's logic, compiled for AArch64, can break. Write to the C++ memory model, not to x86's accidental guarantees, and test on ARM (or with TSan, which models the abstract machine).

<br><br>

---

## Reflection Questions

1. State the two independent guarantees `std::atomic` provides, and give an example using only one of them.
2. Draw the happens-before edges for a release/acquire handshake. Why does the consumer see the producer's plain-data write?
3. Why do both the compiler and CPU reorder memory operations, and why is it invisible single-threaded?
4. When is `relaxed` correct, and when is it a silent bug?
5. Why does a failed `compare_exchange` overwrite `expected`, and how does the CAS loop exploit it?
6. Explain spurious failure and the rule for choosing `weak` vs `strong`.

---

## Interview Questions

1. "Explain the difference between atomicity and memory ordering."
2. "Walk through release/acquire and prove the consumer sees the producer's data."
3. "Define happens-before in terms of sequenced-before and synchronizes-with."
4. "When would you use `memory_order_relaxed`? Give a correct and an incorrect example."
5. "What is `compare_exchange_weak`'s spurious failure, and when do you prefer it over `strong`?"
6. "Why does a failed CAS modify the `expected` argument?"
7. "Implement a lock-free Treiber stack. Justify every memory order you choose."
8. "What is the ABA problem? Why doesn't a plain CAS detect it?"
9. "Your lock-free stack's `pop` calls `delete`. Why is that unsafe, and what fixes it?"
10. "Code passes on x86 with relaxed everywhere but the spec says it's UB. Explain how x86 hides the bug and ARM exposes it."

---

**Next**: Day 32 — Lock-Free Queue →
