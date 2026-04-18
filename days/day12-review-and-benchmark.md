# Day 12: Allocator Review & Benchmark

[← Back to Study Plan](../lld-study-plan.md) | [← Day 11](day11-stl-allocator-interface.md)

> **Time**: ~1.5-2 hours  
> **Goal**: Consolidate everything from Week 2 (Days 8-11). Profile allocator performance, reason about fragmentation trade-offs, and produce hard numbers comparing arena vs pool vs `new`/`delete`.

---
---

# PART 1: WEEK 2 RECAP — ONE PAGE

---
---

<br>

<h2 style="color: #2980B9;">📘 12.1 The Allocator Landscape</h2>

Here is every allocator you've built or studied this week in one table:

| Day | Allocator | Core Mechanism | Alloc | Dealloc | Best For |
|-----|-----------|----------------|-------|---------|----------|
| D8 | `::operator new` / `malloc` | General-purpose heap (free lists, coalescing, headers) | O(n) search | O(n) coalesce | Default; any size, any lifetime |
| D9 | **Arena (bump)** | Increment a pointer | O(1) bump | N/A (bulk `reset()`) | Same-lifetime batches (frames, requests, phases) |
| D10 | **Pool (free-list)** | Pop/push intrusive linked list | O(1) pop | O(1) push | Same-size, independent lifetimes |
| D11 | **STL allocator wrapper** | Forwards to arena or pool | Depends on backing | Depends on backing | Using custom allocators with STL containers |

```
Performance spectrum:

  ←── Fastest                                    Slowest ──→
  Arena (bump)     Pool (free-list)     malloc / new+delete
     ~1-5 ns           ~5-20 ns              ~50-200 ns
```

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 12.2 Memory Layout Comparison</h2>

```
Arena:
┌─────────┬─────────┬─────────┬─────────────────────┐
│  obj A  │  obj B  │  obj C  │      free space      │
└─────────┴─────────┴─────────┴─────────────────────┘
                               ↑ offset
  ✓ Zero fragmentation
  ✓ Perfect cache locality (sequential)
  ✗ Cannot free individual objects


Pool:
┌─────────┬─────────┬─────────┬─────────┬─────────┐
│ ██USED██│ [next→] │ ██USED██│ [next→] │ [null]  │
│  obj A  │  free   │  obj C  │  free   │  free   │
└─────────┴─────────┴─────────┴─────────┴─────────┘
  ✓ Zero external fragmentation (fixed-size → no splitting)
  ✓ Individual free
  ~ Free list becomes shuffled over time (moderate locality)


malloc / new:
┌────┬──────┬────┬───────┬────┬────────────────────┐
│ hdr│obj A │ hdr│ FREE  │ hdr│  obj C  │ ...      │
└────┴──────┴────┴───────┴────┴─────────┴──────────┘
  ✗ Headers add overhead (8-16 bytes per allocation)
  ✗ External fragmentation over time (holes between live objects)
  ✗ Scattered allocations → poor cache locality
  ✓ Fully general: any size, any lifetime
```

<br><br>

---
---

# PART 2: FRAGMENTATION DEEP DIVE

---
---

<br>

<h2 style="color: #2980B9;">📘 12.3 Internal vs External Fragmentation</h2>

| Type | Definition | Example |
|------|-----------|---------|
| **Internal** | Wasted space **inside** an allocated block | Pool block is 32 bytes, but object is 24 → 8 bytes wasted per block |
| **External** | Wasted space **between** allocated blocks (holes) | After many alloc/free cycles, heap has scattered free gaps too small to use |

```
Internal fragmentation (pool):
┌──────────────────────┐
│ object (24 B) │pad 8B│   ← 8 bytes wasted inside the block
└──────────────────────┘
     block = 32 bytes

External fragmentation (malloc after many alloc/free):
┌──────┬──────┬──────┬──────┬──────┬──────┐
│ USED │ FREE │ USED │ FREE │ USED │ FREE │
│ 64B  │ 16B  │ 32B  │ 8B   │ 64B  │ 24B  │
└──────┴──────┴──────┴──────┴──────┴──────┘
  Total free: 48 bytes
  But: cannot satisfy a 48-byte allocation (holes are scattered)!
```

<br>

#### How each allocator handles fragmentation

| Allocator | Internal frag | External frag |
|-----------|---------------|---------------|
| **Arena** | Alignment padding only | **None** — linear, no holes |
| **Pool** | `block_size - sizeof(T)` if T < block | **None** — all blocks are same size, every freed block fits the next allocation |
| **malloc** | Depends on size classes | **Can be severe** — free blocks scatter over time |

The pool's key advantage: because all blocks are the **same size**, a freed block **always** fits the next allocation request. There can never be a "too small to use" hole.

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 12.4 Fragmentation Over Time — Visualized</h2>

```
malloc heap after many random alloc/free (pathological):

Time 1:  [A][B][C][D][E][F][G][H]           ← fully packed
Time 2:  [A][ ][C][ ][E][ ][G][ ]           ← every other freed
Time 3:  [A][I][C][ ][E][J][G][ ]           ← new allocs fill some holes
Time 4:  [ ][I][ ][K][E][J][ ][L]           ← more frees, more holes
         ...
         Over time: many small holes, hard to satisfy large allocations


Arena after many alloc/reset cycles:

Cycle 1: [A][B][C][D][E]          ← sequential, no holes
reset(): [                    ]   ← instantly clean
Cycle 2: [F][G][H]                ← sequential again
         ...
         No fragmentation ever (but no per-object free)


Pool after many alloc/free:

Time 1:  [A][B][C][D][E]          ← sequential
free(B): [A][ ][C][D][E]          ← hole, but it's exactly one block
alloc(): [A][F][C][D][E]          ← hole filled perfectly (same size)
         ...
         No external fragmentation (holes always reusable)
```

<br><br>

---
---

# PART 3: BENCHMARKING

---
---

<br>

<h2 style="color: #2980B9;">📘 12.5 What to Measure</h2>

| Metric | Why it matters |
|--------|---------------|
| **Throughput** | Allocations per second |
| **Latency** | Time per single alloc/dealloc (avg, p99) |
| **Memory overhead** | How much extra memory beyond the objects themselves |
| **Cache behavior** | L1/L2 miss rates when iterating over allocated objects |

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 12.6 Benchmark Patterns</h2>

We'll measure three patterns that stress allocators differently:

<br>

#### Pattern 1: Alloc-only (arena's sweet spot)

```
Allocate N objects sequentially. No frees.
  → Arena: bump pointer N times
  → Pool: pop N blocks
  → new: N heap allocations
```

<br>

#### Pattern 2: Alloc-then-free-all (arena with reset)

```
Allocate N objects, then free all.
  → Arena: bump N, then reset() (one instruction)
  → Pool: pop N, then push N
  → new: N allocations + N deletions
```

<br>

#### Pattern 3: Interleaved alloc/free (pool's sweet spot)

```
Repeatedly: allocate one, use it, free it. N times.
  → Arena: CANNOT do this (no individual free) — must reset all
  → Pool: pop + push per iteration
  → new: new + delete per iteration
```

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 12.7 Complete Benchmark Code</h2>

```cpp
#include <iostream>
#include <chrono>
#include <vector>
#include <cstdlib>
#include <cstring>
#include <new>

// ── Utility ─────────────────────────────────────────────

inline std::size_t align_up(std::size_t n, std::size_t a) {
    return (n + a - 1) & ~(a - 1);
}

// ── Arena Allocator (from Day 9) ────────────────────────

class Arena {
    char* m_buffer;
    std::size_t m_capacity;
    std::size_t m_offset;
public:
    explicit Arena(std::size_t cap)
        : m_buffer(static_cast<char*>(std::malloc(cap)))
        , m_capacity(cap), m_offset(0) {}

    ~Arena() { std::free(m_buffer); }
    Arena(const Arena&) = delete;
    Arena& operator=(const Arena&) = delete;

    void* alloc(std::size_t size, std::size_t alignment) {
        std::size_t aligned = align_up(m_offset, alignment);
        if (aligned + size > m_capacity) return nullptr;
        void* p = m_buffer + aligned;
        m_offset = aligned + size;
        return p;
    }

    template <typename T, typename... Args>
    T* create(Args&&... args) {
        void* p = alloc(sizeof(T), alignof(T));
        return new (p) T(std::forward<Args>(args)...);
    }

    void reset() { m_offset = 0; }
    std::size_t used() const { return m_offset; }
    std::size_t capacity() const { return m_capacity; }
};

// ── Pool Allocator (from Day 10) ────────────────────────

struct FreeBlock { FreeBlock* next; };

template <typename T>
class PoolAllocator {
    char* m_buffer = nullptr;
    FreeBlock* m_freeList = nullptr;
    std::size_t m_blockSize = 0;
    std::size_t m_numBlocks = 0;
public:
    explicit PoolAllocator(std::size_t n) : m_numBlocks(n) {
        m_blockSize = align_up(std::max(sizeof(T), sizeof(FreeBlock)), alignof(T));
        m_buffer = static_cast<char*>(std::malloc(m_blockSize * n));
        m_freeList = nullptr;
        for (std::size_t i = n; i > 0; --i) {
            auto* node = reinterpret_cast<FreeBlock*>(m_buffer + (i - 1) * m_blockSize);
            node->next = m_freeList;
            m_freeList = node;
        }
    }
    ~PoolAllocator() { std::free(m_buffer); }
    PoolAllocator(const PoolAllocator&) = delete;
    PoolAllocator& operator=(const PoolAllocator&) = delete;

    void* allocate() {
        FreeBlock* b = m_freeList;
        m_freeList = b->next;
        return b;
    }
    void deallocate(void* p) {
        auto* b = static_cast<FreeBlock*>(p);
        b->next = m_freeList;
        m_freeList = b;
    }
    template <typename... Args>
    T* create(Args&&... args) {
        return new (allocate()) T(std::forward<Args>(args)...);
    }
    void destroy(T* obj) {
        obj->~T();
        deallocate(obj);
    }
};

// ── Benchmark harness ───────────────────────────────────

struct Obj {
    char data[64];
};

using Clock = std::chrono::high_resolution_clock;

template <typename Fn>
long long measure_us(Fn&& fn) {
    auto start = Clock::now();
    fn();
    auto end = Clock::now();
    return std::chrono::duration_cast<std::chrono::microseconds>(end - start).count();
}

int main() {
    constexpr int N = 1'000'000;

    std::printf("Benchmarking %d allocations of %zu-byte objects\n\n", N, sizeof(Obj));

    // ── Pattern 1: Alloc-only ───────────────────────────

    std::printf("=== Pattern 1: Alloc-only (no free) ===\n");
    {
        long long t_arena = measure_us([&] {
            Arena arena(N * (sizeof(Obj) + 8));
            for (int i = 0; i < N; ++i) arena.create<Obj>();
        });

        long long t_pool = measure_us([&] {
            PoolAllocator<Obj> pool(N);
            for (int i = 0; i < N; ++i) pool.create();
        });

        long long t_new = measure_us([&] {
            std::vector<Obj*> ptrs(N);
            for (int i = 0; i < N; ++i) ptrs[i] = new Obj();
            for (int i = 0; i < N; ++i) delete ptrs[i];
        });

        std::printf("  Arena:      %7lld us\n", t_arena);
        std::printf("  Pool:       %7lld us\n", t_pool);
        std::printf("  new/delete: %7lld us\n", t_new);
    }

    // ── Pattern 2: Alloc-then-free-all ──────────────────

    std::printf("\n=== Pattern 2: Alloc N, then free all ===\n");
    {
        long long t_arena = measure_us([&] {
            Arena arena(N * (sizeof(Obj) + 8));
            for (int i = 0; i < N; ++i) arena.create<Obj>();
            arena.reset();
        });

        long long t_pool = measure_us([&] {
            PoolAllocator<Obj> pool(N);
            std::vector<Obj*> ptrs(N);
            for (int i = 0; i < N; ++i) ptrs[i] = pool.create();
            for (int i = 0; i < N; ++i) pool.destroy(ptrs[i]);
        });

        long long t_new = measure_us([&] {
            std::vector<Obj*> ptrs(N);
            for (int i = 0; i < N; ++i) ptrs[i] = new Obj();
            for (int i = 0; i < N; ++i) delete ptrs[i];
        });

        std::printf("  Arena:      %7lld us\n", t_arena);
        std::printf("  Pool:       %7lld us\n", t_pool);
        std::printf("  new/delete: %7lld us\n", t_new);
    }

    // ── Pattern 3: Interleaved alloc/free ───────────────

    std::printf("\n=== Pattern 3: Interleaved alloc + free (1 at a time) ===\n");
    {
        long long t_pool = measure_us([&] {
            PoolAllocator<Obj> pool(1);
            for (int i = 0; i < N; ++i) {
                Obj* o = pool.create();
                pool.destroy(o);
            }
        });

        long long t_new = measure_us([&] {
            for (int i = 0; i < N; ++i) {
                Obj* o = new Obj();
                delete o;
            }
        });

        std::printf("  Arena:      N/A (cannot free individual objects)\n");
        std::printf("  Pool:       %7lld us\n", t_pool);
        std::printf("  new/delete: %7lld us\n", t_new);
    }

    std::printf("\nDone.\n");
    return 0;
}
```

<br>

### Compile and run

```bash
g++ -std=c++17 -O2 -Wall -Wextra -o day12_bench day12_bench.cpp && ./day12_bench
```

**Important**: Use `-O2` (or `-O3`) so the compiler doesn't optimize away dead code, but also doesn't leave debug overhead. Don't use `-fsanitize=address` for benchmarking — it adds ~2x overhead.

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 12.8 Expected Results (Approximate)</h2>

Results vary by CPU, OS, compiler, and heap implementation. Typical order of magnitude on a modern x86-64 laptop:

```
Pattern 1: Alloc-only (1M × 64-byte objects)
  Arena:        2,000 -   5,000 us    ← fastest (just bump)
  Pool:         5,000 -  15,000 us    ← fast (pop free list)
  new/delete:  80,000 - 200,000 us    ← slowest (heap bookkeeping)

Pattern 2: Alloc N, then free all
  Arena:        2,000 -   5,000 us    ← reset() is nearly free
  Pool:        10,000 -  30,000 us    ← N pops + N pushes
  new/delete:  80,000 - 200,000 us    ← N allocs + N frees

Pattern 3: Interleaved alloc/free (1 at a time)
  Arena:        N/A
  Pool:         3,000 -   8,000 us    ← one pop + one push per iter
  new/delete:  50,000 - 150,000 us    ← full heap round-trip per iter
```

**Key takeaways:**
- Arena is **10-50x** faster than `new`/`delete` for batch workloads
- Pool is **10-30x** faster for per-object alloc/free
- The gap widens with larger objects (more bookkeeping for `malloc`)
- Cache locality is a secondary but significant factor (not measured by raw time alone)

<br><br>

---
---

# PART 4: UNDERSTANDING THE NUMBERS

---
---

<br>

<h2 style="color: #2980B9;">📘 12.9 Where Does <code>new</code>'s Time Go?</h2>

When you call `new Obj()`, the runtime does:

```
1. Enter ::operator new(sizeof(Obj))
2.   → calls malloc(sizeof(Obj))
3.      → lock the heap (or use thread-local cache)
4.      → search size-class bins for a free block
5.      → if no block: request memory from OS (sbrk/mmap)
6.      → split block if oversized
7.      → write header/footer metadata
8.      → return pointer
9. Back in operator new: return pointer
10. Compiler emits constructor call on returned memory
```

Steps 3-7 are where the time goes. Arena skips **all** of them (just increment offset). Pool skips steps 3-7 (just read `next` pointer).

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 12.10 Cache Effects — The Hidden Factor</h2>

Raw alloc/dealloc benchmarks don't show the full picture. The **real** performance difference appears when you **iterate over** allocated objects:

```cpp
// After allocating N objects, iterate and touch each one:
long long t_iterate = measure_us([&] {
    volatile int sum = 0;
    for (int i = 0; i < N; ++i) {
        sum += ptrs[i]->data[0];   // touches one cache line per object
    }
});
```

With **contiguous** allocation (arena/pool), objects are packed in cache lines → prefetcher works well → fast.

With **scattered** `new` allocations, each object might be in a different cache line or even a different page → cache misses → slow.

```
Iteration over 1M objects (approximate):
  Arena-allocated (contiguous):     1,000 -  3,000 us
  Pool-allocated (mostly contiguous): 2,000 -  5,000 us
  new-allocated (scattered):       10,000 - 30,000 us
```

This is why game engines use arenas/pools for hot data — the iteration speedup is often **more important** than the allocation speedup.

<br><br>

---
---

# PART 5: OVERHEAD ANALYSIS

---
---

<br>

<h2 style="color: #2980B9;">📘 12.11 Memory Overhead per Object</h2>

| Allocator | Overhead per object | Total for 1M × 64B objects |
|-----------|--------------------|-----------------------------|
| **Arena** | 0 bytes (only alignment padding, typically ≤7B) | ~64 MB (near-zero waste) |
| **Pool** | `block_size - sizeof(T)` (alignment padding only) | ~64 MB |
| **`malloc`** | 8-16 bytes header per allocation | 72-80 MB (12-25% overhead) |

For small objects (`sizeof(T)` = 8-16 bytes), `malloc`'s per-object header can be **50-100% overhead**. That's where custom allocators shine the most.

```
malloc overhead for different object sizes:

  sizeof(T) = 8   + 16 header = 24 bytes total  → 200% overhead
  sizeof(T) = 16  + 16 header = 32 bytes total  → 100% overhead
  sizeof(T) = 64  + 16 header = 80 bytes total  →  25% overhead
  sizeof(T) = 256 + 16 header = 272 bytes total →   6% overhead
  sizeof(T) = 4096 + 16 header = 4112 total     →  <1% overhead

→ Custom allocators matter most for small, frequently allocated objects
```

<br><br>

---
---

# PART 6: DECISION GUIDE — WHICH ALLOCATOR WHEN?

---
---

<br>

<h2 style="color: #2980B9;">📘 12.12 Flowchart</h2>

```
Start: "I need to allocate objects"
  │
  ├── All objects same lifetime? (batch/phase/frame)
  │     YES → Arena
  │       └── Need to iterate over them? → Arena (best cache locality)
  │
  ├── All objects same size?
  │     YES → Individual free needed?
  │             YES → Pool
  │             NO  → Arena
  │
  ├── Variable sizes + individual free needed?
  │     YES → malloc / new (or multi-size pool)
  │
  ├── STL containers involved?
  │     YES → Need same type across different allocators?
  │             YES → std::pmr (C++17)
  │             NO  → Template allocator wrapper (Day 11)
  │
  └── Don't know yet?
        → Start with new/delete, profile, then replace hot paths
```

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 12.13 Real-World Combinations</h2>

Production systems rarely use just one allocator. Common patterns:

| System | Allocation Strategy |
|--------|--------------------|
| **Game engine** | Frame arena (per-frame temps) + pool per component type + `malloc` fallback for rare large allocs |
| **Compiler** | Arena per compilation phase (lexer, parser, optimizer each get their own arena) |
| **Web server** | Per-request arena (allocate during request, reset at response end) |
| **Database** | Buffer pool (fixed-size pages) + arena for query-plan temporaries |
| **Embedded** | Static pools for every object type (no `malloc` at all — deterministic memory usage) |

<br><br>

---
---

# PART 7: PROFILING TOOLS

---
---

<br>

<h2 style="color: #2980B9;">📘 12.14 Tools You Should Know</h2>

<br>

#### AddressSanitizer (`-fsanitize=address`)

Detects: buffer overflows, use-after-free, double-free, memory leaks.

```bash
g++ -std=c++17 -fsanitize=address -g -o test test.cpp && ./test
```

Adds ~2x runtime overhead. Use for correctness testing, **not** benchmarking.

<br>

#### Valgrind / Memcheck

```bash
valgrind --leak-check=full ./test
```

Detects similar issues to ASan but runs ~10-20x slower. Works without recompilation. Shows exact leak locations.

<br>

#### Valgrind Massif (heap profiler)

```bash
valgrind --tool=massif ./test
ms_print massif.out.<pid>
```

Shows heap usage over time — peaks, growth patterns, which call sites allocate the most. Useful for finding where custom allocators would help.

<br>

#### `perf stat` (Linux — hardware counters)

```bash
perf stat -e cache-misses,cache-references,instructions,cycles ./test
```

Shows cache miss rates. Compare arena-allocated iteration vs `new`-allocated iteration — you'll see dramatically fewer cache misses with the arena.

<br>

#### Custom `operator new` logging (Day 8)

Your Day 8 allocation tracker is a lightweight profiling tool: count allocations, total bytes, peak usage, without external tools.

<br><br>

---
---

# PART 8: BUILD EXERCISE

---
---

<br>

<h2 style="color: #2980B9;">📘 12.15 Exercise: Write the Full Benchmark</h2>

Take the benchmark code from section 12.7 and extend it:

1. **Add iteration benchmark**: After allocating N objects, iterate and touch each one. Measure the iteration time separately to see cache effects.

2. **Add object size sweep**: Run the benchmark for `sizeof(Obj)` = 8, 32, 64, 256, 1024. Print a table showing how the speedup ratio changes with object size.

3. **Add memory overhead measurement**: Track total bytes requested vs total bytes used by each allocator. Print the overhead percentage.

4. **Print results as a formatted table**:

```
Size    Pattern    Arena(us)  Pool(us)  new(us)  Arena_speedup  Pool_speedup
───────────────────────────────────────────────────────────────────────────
   8    alloc-only    1200      3500     95000        79x           27x
  64    alloc-only    2100      5800    120000        57x           21x
 256    alloc-only    4500      9200    135000        30x           15x
```

<br>

### Compile and run

```bash
g++ -std=c++17 -O2 -Wall -Wextra -o day12 day12_bench.cpp && ./day12
```

<br>

#### Bonus Challenges

1. **p99 latency**: Instead of total time, measure the worst-case single allocation time. Use `std::vector<long long>` to store per-alloc times, sort, and print p50/p99/p99.9. You'll see `malloc` has occasional spikes (OS interactions) while arena/pool are consistent.

2. **Multi-threaded benchmark**: Allocate from N threads simultaneously. Compare: (a) `new` with heap lock contention, (b) `thread_local` arenas, (c) atomic pool (Treiber stack). Measure how contention scales.

3. **Fragmentation stress test**: Allocate 1M objects, free every other one, then try to allocate a large contiguous block. Observe how `malloc` handles (or fails to handle) fragmentation vs pool/arena.

4. **`perf stat` comparison**: Run each pattern under `perf stat` and record cache-miss counts. Compare the iteration benchmark across allocators.

<br><br>

---
---

# PART 9: DEEP DIVE — COMMON QUESTIONS

---
---

<br>

<h2 style="color: #2980B9;">📘 12.16 Q&A</h2>

<br>

#### Q1: "Is `malloc` really that slow?"

Modern `malloc` implementations (glibc, jemalloc, tcmalloc, mimalloc) are heavily optimized with thread-local caches, size-class bins, and clever data structures. They're **much** faster than textbook `malloc`. But they still can't beat a bump pointer (arena) or a single linked-list pop (pool) because they must handle the **general case** — any size, any order, thread safety. The specialization advantage of arena/pool is real and measurable.

<br>

#### Q2: "When is the overhead of `new` not worth optimizing?"

When allocations are **rare** relative to the work done per object. If you allocate a database connection once and use it for millions of queries, the allocation cost is irrelevant. Custom allocators matter when you allocate **thousands to millions** of objects per second (game entities, network packets, parser nodes).

<br>

#### Q3: "Should I always replace `new` with a custom allocator?"

No. `new`/`delete` is the safe default. Custom allocators add complexity (lifetime management, debugging difficulty, code coupling). Only optimize allocation when **profiling shows** it's a bottleneck. The usual progression:

```
1. Use new/delete (correct, simple)
2. Profile → find hot allocation path
3. Replace with arena or pool for that specific path
4. Keep new/delete everywhere else
```

<br>

#### Q4: "Why does the arena benchmark include the arena constructor time?"

In the benchmark above, arena construction (`malloc` for the backing buffer) is included in the measurement. In real usage, the arena is typically created **once** (at startup or scope entry) and reused across many cycles. The per-cycle cost is just `reset()` + bumps — even faster than shown.

<br>

#### Q5: "How do jemalloc / tcmalloc compare?"

Both use **size-class pools internally** with thread-local caches — they're essentially multi-size pool allocators with sophisticated memory management. They're ~2-5x faster than glibc `malloc` for many workloads. But they still can't match a purpose-built arena or pool because they maintain generality (any size, any thread, any lifetime).

```
Performance spectrum (rough):

  Arena       Pool      tcmalloc/jemalloc    glibc malloc
   1x         2-5x          10-30x              30-100x
  (fastest)                                    (slowest)
```

<br><br>

---

## Reflection Questions

1. Which benchmark pattern best matches your current project's allocation behavior?
2. Where would you place custom allocators in a codebase you work on?
3. What is the relationship between object size and allocator overhead?
4. Why is cache locality often more important than raw allocation speed?
5. When should you stop optimizing allocations?

---

## Interview Questions

1. "Compare arena, pool, and general-purpose allocators. When would you use each?"
2. "What is the difference between internal and external fragmentation?"
3. "How would you benchmark a custom allocator against `new`/`delete`?"
4. "Why does cache locality matter for allocation patterns?"
5. "When is optimizing allocation not worth the effort?"
6. "How do jemalloc/tcmalloc improve over glibc malloc?"
7. "Draw the memory layout of an arena vs a pool after 5 allocations and 2 frees."
8. "What profiling tools would you use to find allocation bottlenecks?"

---

**Next**: Day 13 — Object Pool Pattern →
