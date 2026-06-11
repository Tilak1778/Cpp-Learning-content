# Day 34: Parallel Algorithms

[← Back to Study Plan](../lld-study-plan.md) | [← Day 33](day33-actor-model.md)

> **Time**: ~2-3 hours (weekend day)
> **Goal**: You have a thread pool (Days 29-30); now use it to make real work go faster. Learn the **map-reduce** pattern, parallel `for_each`, the difference between **static and dynamic chunking** (work partitioning), how **false sharing** silently destroys parallel speedup, **Amdahl's Law** (why you don't get N× from N cores), and the C++17 `std::execution` policies (overview). Build a **parallel word-frequency counter** that partitions text into chunks, maps each chunk to a local histogram, and reduces the partials into one — using the thread pool.

---
---

# PART 1: MAP-REDUCE & PARTITIONING

---
---

<br>

<h2 style="color: #2980B9;">📘 34.1 The Map-Reduce Shape</h2>

A vast class of data-parallel problems fit one shape:

1. **Map** — split the input into independent chunks; apply a function to each chunk *in parallel*.
2. **Reduce** — combine the per-chunk partial results into one final result.

```
            input: [ ──────────────── data ──────────────── ]
                          │ partition
        ┌───────────┬───────────┬───────────┬───────────┐
        │  chunk 0  │  chunk 1  │  chunk 2  │  chunk 3  │
        └─────┬─────┴─────┬─────┴─────┬─────┴─────┬─────┘
       map ▼          ▼          ▼          ▼        (parallel, on the pool)
        partial0   partial1   partial2   partial3
              └────────┬─────────┬────────┘
                  reduce ▼   (combine partials)
                      final result
```

The magic requirement: **the map step must be embarrassingly parallel** — each chunk processed independently, no shared mutable state. The *only* coordination is the final reduce. Word counting fits perfectly: each chunk produces its own local histogram (no contention), then the partials are merged.

The anti-pattern is having all threads `++sharedHistogram[word]` — that's a contended write to shared memory on every word, requiring a lock or atomic per increment, and it serializes the very thing you parallelized. **Map to local state, reduce at the end.**

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 34.2 Static vs Dynamic Chunking</h2>

How do you split N items across P workers? Two strategies, and the choice depends on whether items take *uniform* time.

#### Static chunking

Divide the range into P equal contiguous chunks up front; each worker gets one.

```
N=1000 items, P=4 workers:
  worker0: [0,   250)
  worker1: [250, 500)
  worker2: [500, 750)
  worker3: [750, 1000)
```

| Pros | Cons |
|------|------|
| Zero scheduling overhead — assign once | **Load imbalance** if items vary in cost |
| Great cache locality (contiguous) | A slow chunk leaves other cores idle waiting |
| No synchronization during the map | The whole job is as slow as the slowest chunk |

#### Dynamic chunking (work queue)

Split into *many small* chunks (more than P); workers pull the next chunk when they finish one — exactly the thread-pool model.

```
N=1000, chunk size 50 → 20 chunks; 4 workers grab chunks as they free up:
  fast workers process more chunks; slow ones fewer → auto-balanced
```

| Pros | Cons |
|------|------|
| Self-balancing — idle workers steal more work | Per-chunk scheduling/queue overhead |
| Robust to uneven item costs | Smaller chunks = worse locality + more overhead |
| Maps naturally onto a thread pool / work stealing | Need to tune chunk size |

**Rule of thumb**: uniform-cost items → static (or coarse) chunking. Highly variable costs → dynamic with a chunk size tuned so per-chunk overhead is amortized but imbalance stays small (often "a few × P" chunks, or a grain size of thousands of cheap items). This is the static-vs-dynamic loop-scheduling tradeoff from OpenMP.

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 34.3 Amdahl's Law — Why Not N×?</h2>

If a fraction `s` of your program is inherently **serial** (can't be parallelized — e.g., the reduce, I/O, setup), the maximum speedup with P processors is bounded:

```
            1
Speedup ≤ ─────────────         (Amdahl's Law)
          s + (1 - s)/P

Example: s = 5% serial, P = 16 cores
   Speedup ≤ 1 / (0.05 + 0.95/16) = 1 / 0.109 ≈ 9.1×   (not 16×!)

   As P → ∞:  Speedup → 1/s = 20×   (a hard ceiling from 5% serial)
```

The lesson: a small serial fraction caps your speedup hard. 5% serial means you can *never* beat 20×, no matter how many cores. So:
- Minimize the serial portion (overlap I/O, parallelize the reduce as a tree).
- Don't expect linear scaling; measure actual speedup, not core count.
- Beware **Gustafson's Law's** counterpoint: if you scale the *problem size* with cores, the serial fraction shrinks proportionally, so big-data workloads scale better than Amdahl suggests for fixed-size problems.

<br><br>

---
---

# PART 2: FALSE SHARING & EXECUTION POLICIES

---
---

<br>

<h2 style="color: #2980B9;">📘 34.4 False Sharing in Parallel Reductions</h2>

A subtle performance killer (we met it in the SPSC buffer, Day 32). Suppose each worker accumulates into a shared array, one slot per worker:

```cpp
std::vector<long> partials(numWorkers, 0);    // BAD layout
// worker i does:  partials[i] += ...;          // many times
```

Even though each worker writes a *different* index (no logical conflict), adjacent `long`s share a 64-byte cache line — so every write by worker 0 invalidates worker 1's cached copy and vice versa. The cores bounce the line back and forth (cache-line ping-pong). Result: a "parallel" loop that's *slower* than serial.

```
partials[0..7] all in ONE cache line:
   write by core0 → line dirty in core0, INVALIDATED in core1..7
   write by core1 → line bounces to core1, invalidated elsewhere
   ... thrash ...
```

Fixes:
- **Local accumulator**: each worker accumulates into a *stack-local* variable and writes the shared slot **once** at the end. (Best — what map-reduce naturally does.)
- **Pad each slot** to its own cache line (`alignas(hardware_destructive_interference_size)`).
- **Per-thread containers** reduced at the end (our word-count approach: each chunk owns its own `unordered_map`).

False sharing is invisible in the code — it's a *layout* property — which is why it's a favourite interview "why is my parallel code slow?" question. The answer is almost always "the threads are writing to the same cache line; give them independent storage."

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 34.5 C++17 Execution Policies (Overview)</h2>

C++17 added overloads of many `<algorithm>` functions that take an **execution policy** as the first argument, letting the standard library parallelize for you:

```cpp
#include <execution>
#include <algorithm>
#include <numeric>

std::for_each(std::execution::par, v.begin(), v.end(), work);          // parallel
std::sort(std::execution::par_unseq, v.begin(), v.end());              // parallel + vectorized
long sum = std::reduce(std::execution::par, v.begin(), v.end(), 0L);   // parallel reduce
double tot = std::transform_reduce(std::execution::par,                // map-reduce in one call!
                v.begin(), v.end(), 0.0, std::plus<>{},
                [](const auto& x){ return cost(x); });
```

| Policy | Meaning |
|--------|---------|
| `seq` | Sequential, in order (the classic single-threaded behaviour) |
| `par` | May run on multiple threads; element operations must not data-race |
| `par_unseq` | Parallel **and** vectorized (SIMD); operations may also be *interleaved* within a thread — so they must be **vectorization-safe** (no locks, no allocation that could deadlock, no order dependence) |
| `unseq` (C++20) | Vectorized on one thread |

Caveats: `par`/`par_unseq` give *no ordering guarantees*; your element function must be free of data races (for `par`) and additionally free of inter-element dependencies and lock acquisition (for `par_unseq`, since SIMD lanes can't take locks). `transform_reduce` is literally map-reduce as a one-liner — prefer it when it fits. Library support varies (GCC needs Intel TBB linked: `-ltbb`); the policies are a *request*, and an implementation may run them sequentially. Knowing they exist — and that we're hand-building what they automate — is the point today.

<br><br>

---
---

# PART 3: BUILD EXERCISE

---
---

<br>

<h2 style="color: #2980B9;">📘 34.6 Exercise: Parallel Word-Frequency Counter</h2>

Count word frequencies in a large block of text, in parallel, using the Day 29 thread pool. Partition the text into chunks (respecting word boundaries), **map** each chunk to a local `unordered_map<string,size_t>` (no shared state, no locks during the map), then **reduce** the partial maps into one. Verify the parallel result matches a serial baseline exactly.

This combines: the thread pool (`submit` → `future`), map-reduce, static chunking with boundary fixup, and per-thread local state to avoid false sharing/contention.

<br>

#### Skeleton

```cpp
// parallel_wordcount.h
#pragma once
#include "thread_pool.h"     // the Day 29 ThreadPool: submit<F>(...) -> future
#include <cctype>
#include <future>
#include <string>
#include <string_view>
#include <unordered_map>
#include <vector>

using Histogram = std::unordered_map<std::string, std::size_t>;

// --- normalize: lowercase, split on non-alphanumeric ---
inline void countInto(std::string_view text, Histogram& hist) {
    std::string word;
    auto flush = [&] {
        if (!word.empty()) { ++hist[word]; word.clear(); }
    };
    for (char c : text) {
        if (std::isalnum(static_cast<unsigned char>(c)))
            word.push_back(static_cast<char>(std::tolower(static_cast<unsigned char>(c))));
        else
            flush();
    }
    flush();   // trailing word
}

// --- serial baseline (for verification) ---
inline Histogram countSerial(std::string_view text) {
    Histogram h;
    countInto(text, h);
    return h;
}

// --- partition the text into N chunks on WORD boundaries ---
// Returns [begin,end) offsets; each boundary is advanced to the next
// whitespace so no word is split across two chunks.
inline std::vector<std::pair<std::size_t, std::size_t>>
partition(std::string_view text, std::size_t chunks) {
    std::vector<std::pair<std::size_t, std::size_t>> ranges;
    const std::size_t n = text.size();
    if (n == 0 || chunks == 0) return ranges;
    const std::size_t approx = n / chunks;

    std::size_t start = 0;
    for (std::size_t c = 0; c < chunks && start < n; ++c) {
        std::size_t end = (c + 1 == chunks) ? n : std::min(n, start + approx);
        // extend 'end' to the next non-alphanumeric so we don't split a word
        while (end < n && std::isalnum(static_cast<unsigned char>(text[end])))
            ++end;
        ranges.emplace_back(start, end);
        start = end;
    }
    return ranges;
}

// --- parallel map-reduce word count ---
inline Histogram countParallel(std::string_view text, ThreadPool& pool,
                               std::size_t chunks) {
    auto ranges = partition(text, chunks);

    // MAP: each chunk -> its own local Histogram (no shared state, no locks).
    std::vector<std::future<Histogram>> futures;
    futures.reserve(ranges.size());
    for (auto [b, e] : ranges) {
        futures.push_back(pool.submit([text, b, e] {
            Histogram local;
            countInto(text.substr(b, e - b), local);
            return local;        // returned by value through the future
        }));
    }

    // REDUCE: merge partials into one. (Serial reduce here — see Bonus 4 for
    // a parallel tree-reduce.)
    Histogram total;
    for (auto& f : futures) {
        Histogram part = f.get();
        for (auto& [word, count] : part)
            total[word] += count;
    }
    return total;
}
```

<br>

#### Test driver

```cpp
// main.cpp
#include "parallel_wordcount.h"
#include <cassert>
#include <chrono>
#include <iostream>
#include <random>
#include <sstream>

// Build a big synthetic text with a known vocabulary.
static std::string makeText(std::size_t words) {
    static const char* vocab[] = {
        "alpha","beta","gamma","delta","alpha","epsilon","beta","alpha","zeta","beta"
    };
    std::mt19937 rng(12345);
    std::uniform_int_distribution<int> pick(0, 9);
    std::ostringstream os;
    for (std::size_t i = 0; i < words; ++i)
        os << vocab[pick(rng)] << ((i % 13 == 12) ? "\n" : " ");
    return os.str();
}

int main() {
    std::cout << "=== 1. Correctness: parallel == serial ===\n";
    {
        std::string text =
            "The quick brown fox. The lazy dog! Quick quick FOX fox fox.";
        Histogram serial = countSerial(text);
        ThreadPool pool(4);
        Histogram par = countParallel(text, pool, 4);

        assert(serial == par);             // exact match including all counts
        std::cout << "  'fox'   = " << par["fox"]   << " (expect 4)\n";
        std::cout << "  'quick' = " << par["quick"] << " (expect 3)\n";
        std::cout << "  'the'   = " << par["the"]   << " (expect 2)\n";
        assert(par["fox"] == 4);
        assert(par["quick"] == 3);
        assert(par["the"] == 2);
    }

    std::cout << "=== 2. Large input: counts conserved, speedup ===\n";
    {
        std::string text = makeText(2'000'000);   // ~2M words

        auto t0 = std::chrono::steady_clock::now();
        Histogram serial = countSerial(text);
        auto t1 = std::chrono::steady_clock::now();

        ThreadPool pool;                           // hardware_concurrency workers
        Histogram par = countParallel(text, pool, pool.size() * 4);  // dynamic-ish
        auto t2 = std::chrono::steady_clock::now();

        assert(serial == par);

        std::size_t totalSerial = 0, totalPar = 0;
        for (auto& [w, c] : serial) totalSerial += c;
        for (auto& [w, c] : par)    totalPar += c;
        assert(totalSerial == totalPar);

        auto ms = [](auto a, auto b) {
            return std::chrono::duration_cast<std::chrono::milliseconds>(b - a).count();
        };
        std::cout << "  total words  = " << totalPar << "\n";
        std::cout << "  serial   : " << ms(t0, t1) << " ms\n";
        std::cout << "  parallel : " << ms(t1, t2) << " ms  (workers="
                  << pool.size() << ")\n";
    }

    std::cout << "All assertions passed.\n";
    return 0;
}
```

<br>

#### Compile and run

```bash
# Functional correctness under ThreadSanitizer:
g++ -std=c++17 -Wall -Wextra -Wpedantic -fsanitize=thread -pthread \
    -o day34_tsan main.cpp && ./day34_tsan

# Performance run (optimized, no sanitizer overhead):
g++ -std=c++17 -Wall -Wextra -Wpedantic -O2 -pthread \
    -o day34 main.cpp && ./day34
```

(`thread_pool.h` is the Day 29 header. ThreadSanitizer should report **no** races — the map phase touches only thread-local histograms, and results travel home through futures.)

<br>

#### Expected output (timings will vary)

```
=== 1. Correctness: parallel == serial ===
  'fox'   = 4 (expect 4)
  'quick' = 3 (expect 3)
  'the'   = 2 (expect 2)
=== 2. Large input: counts conserved, speedup ===
  total words  = 2000000
  serial   : 210 ms
  parallel : 64 ms  (workers=8)
All assertions passed.
```

<br>

#### Bonus Challenges

1. **Static vs dynamic chunking experiment** — run with `chunks == pool.size()` (static) and `chunks == pool.size() * 16` (finer/dynamic). Make some chunks artificially expensive (e.g., sleep proportional to a hash) and observe how dynamic chunking balances better.

2. **Demonstrate false sharing** — implement a *wrong* reducer where workers `++shared[bucket]` into a shared array of adjacent counters under no padding; benchmark against the local-histogram version and explain the slowdown.

3. **Parallel reduce (tree)** — the serial merge of partials is itself `O(chunks)`. Implement a tree reduction (pairwise merge across `log(chunks)` levels, each level on the pool) and measure whether it helps at high chunk counts (Amdahl: shrinking the serial reduce).

4. **`std::transform_reduce` comparison** — re-express a numeric version (e.g., total character count) with `std::transform_reduce(std::execution::par, ...)` and compare LOC, performance, and portability (`-ltbb` on GCC) against the hand-rolled pool version.

5. **Real file + mmap** — read an actual large text file (mmap it for zero-copy), partition the mapped region, and count. Handle UTF-8 word boundaries at least naively.

6. **Top-K words** — after reduction, find the K most frequent words. Do it with a partial sort / heap and discuss whether the top-K step is worth parallelizing (usually not — it's tiny vs the count).

<br><br>

---
---

# PART 4: DEEP DIVE — COMMON QUESTIONS

---
---

<br>

<h2 style="color: #2980B9;">📘 34.7 Q&A</h2>

<br>

#### Q1: "Why map to local state instead of a shared accumulator?"

A shared accumulator (one `unordered_map` all threads write) forces a lock or atomic on *every* word — serializing the hot loop and adding contention/false sharing, often making the parallel version slower than serial. Mapping to per-chunk local histograms means the parallel map phase has **zero** synchronization; all coordination is deferred to a single cheap reduce at the end. This is the defining discipline of map-reduce.

<br>

#### Q2: "Static or dynamic chunking — how do I choose?"

If every item costs roughly the same, static (equal contiguous chunks) wins: no scheduling overhead, great cache locality. If item costs vary widely, static causes load imbalance (the slowest chunk gates the whole job), so use dynamic — many small chunks pulled from a queue so idle workers grab more. Tune chunk size so per-chunk overhead is small relative to chunk work but chunks stay numerous enough to balance (a common heuristic: several × the worker count).

<br>

#### Q3: "Why don't I get N× speedup on N cores?"

Amdahl's Law: any serial fraction `s` caps speedup at `1/(s + (1-s)/P)`, approaching `1/s` as cores grow. Setup, the reduce step, I/O, and memory bandwidth limits are all serial or shared bottlenecks. Plus real overheads: thread scheduling, cache coherence traffic, and false sharing. Measure actual speedup; treat "N cores → N×" as a fairy tale.

<br>

#### Q4: "What is false sharing and how does it ruin parallel reductions?"

When threads write to *different* variables that happen to live on the *same* cache line, each write invalidates the others' cached copy, causing the line to ping-pong between cores. The threads don't logically conflict, but the hardware coherence protocol serializes them anyway. In reductions it appears when per-thread counters sit adjacent in an array. Fix: accumulate in thread-local variables and write the shared slot once, or pad each slot to its own cache line.

<br>

#### Q5: "What do `seq`, `par`, and `par_unseq` actually require of my callable?"

`seq`: nothing special (ordered, single-threaded). `par`: your element operation may run on multiple threads, so it must be **data-race-free** (no unsynchronized shared writes). `par_unseq`: additionally the operations may be **interleaved/vectorized within a single thread**, so they must not acquire locks, must not depend on element order, and must be safe to execute as SIMD lanes — taking a mutex inside a `par_unseq` body can deadlock. Choose the weakest policy whose constraints your code satisfies.

<br>

#### Q6: "Why does partitioning need to respect word boundaries?"

If you split the byte range naively, a chunk boundary can land in the middle of a word ("paral|lel"), so two chunks would count two fake fragments instead of one real word — the parallel result wouldn't match serial. Advancing each boundary forward to the next non-alphanumeric character ensures every word lands wholly in exactly one chunk, making the parallel count identical to the serial one.

<br>

#### Q7: "Is the serial reduce a bottleneck?"

It can be at high chunk counts — merging `C` partial maps is `O(C × distinct_words)` and runs on one thread, contributing to Amdahl's serial fraction. For modest `C` it's negligible. When it matters, parallelize it as a **tree reduction**: merge pairs of partials across `log(C)` levels, each level's merges running on the pool. Whether that helps depends on map cost vs reduce cost — measure.

<br>

#### Q8: "When should I just use `std::execution::par` instead of a hand-rolled pool?"

When the algorithm fits a standard one (`for_each`, `reduce`, `transform_reduce`, `sort`) and you don't need custom scheduling, the policy version is shorter, well-tested, and may use TBB's work-stealing scheduler. Hand-roll on a pool when you need control over chunking, custom reduce shapes, integration with your own pool/priorities, or your platform's policy support is weak. Knowing both lets you reach for the library first and drop down only when needed.

<br><br>

---

## Reflection Questions

1. Describe the map-reduce shape. What property must the map step have for it to parallelize cleanly?
2. Why is mapping to local state and reducing at the end superior to a shared accumulator?
3. Compare static and dynamic chunking. When does each win, and what's the cost of getting it wrong?
4. State Amdahl's Law and explain why 5% serial work caps you near 20× regardless of cores.
5. What is false sharing, why is it invisible in the source, and how does the word counter avoid it?
6. What are the constraints `par` vs `par_unseq` place on your element function, and why is a lock unsafe under `par_unseq`?

---

## Interview Questions

1. "Design a parallel word-frequency counter. How do you partition, map, and reduce?"
2. "Why count into per-chunk local maps instead of one shared map?"
3. "Static vs dynamic chunking — explain the tradeoff and when you'd pick each."
4. "Derive Amdahl's Law and use it: 10% serial, 32 cores — what's the max speedup?"
5. "Your parallel reduction is slower than serial. What's the most likely cause and how do you confirm it?"
6. "Explain `std::execution::seq`, `par`, and `par_unseq` and the requirements each imposes."
7. "Why must chunk boundaries respect word boundaries, and how do you implement that?"
8. "When is the reduce step a bottleneck, and how do you parallelize it?"
9. "How would you express this with `std::transform_reduce`, and what are the portability caveats?"
10. "How do you find the top-K most frequent words after counting, and is that step worth parallelizing?"

---

**Next**: Day 35 — Weekly Review →
