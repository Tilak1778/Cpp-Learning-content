# Day 10: Pool (Free-List) Allocator

[← Back to Study Plan](../lld-study-plan.md) | [← Day 9](day09-arena-allocator.md)

> **Time**: ~1.5-2 hours
> **Goal**: Understand fixed-size pool allocation, implement an intrusive free-list allocator from scratch, and benchmark it against `new`/`delete` and your Day 9 arena.

---
---

# PART 1: THE PROBLEM POOLS SOLVE

---
---

<br>

<h2 style="color: #2980B9;">📘 10.1 When Arenas Aren't Enough</h2>

Day 9's arena is blazing fast — but it has one hard constraint: **you cannot free individual objects.** You must free everything at once with `reset()`.

Many real workloads need **per-object lifetime**:

| Scenario | Why arena fails |
|----------|----------------|
| Game entity system | Enemies spawn and die at random times — you can't wait for "end of frame" to free |
| Network connections | Connections open/close independently |
| AST nodes during optimization | Some nodes are deleted while neighbors survive |
| Message queues | Messages are consumed (freed) one at a time |

You could fall back to `new`/`delete`, but those are slow (Day 8: search free lists, coalesce, bookkeeping). What if all your objects are the **same size**?

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 10.2 The Key Insight: Same-Size Objects</h2>

When every allocation is the **exact same size**, most of `malloc`'s complexity disappears:

```
malloc's problems:
  ✗ Search for a fitting block  → NOT needed: every block fits
  ✗ Split large blocks          → NOT needed: blocks are pre-sized
  ✗ Coalesce adjacent blocks    → NOT needed: all blocks are same size
  ✗ Variable-size bookkeeping   → NOT needed: fixed layout
```

**A pool allocator** pre-divides a buffer into **fixed-size chunks** and maintains a **free list** — a linked list of available chunks. Allocation and deallocation are both **O(1)**: pop from or push to the free list.

<br><br>

---
---

# PART 2: HOW A POOL ALLOCATOR WORKS

---
---

<br>

<h2 style="color: #2980B9;">📘 10.3 The Free List — Core Idea</h2>

A pool starts with a contiguous buffer chopped into N **blocks** of equal size. Each free block stores a **pointer to the next free block** inside its own memory (since it's free, nobody else is using those bytes). This is called an **intrusive free list**.

```
Initial state (pool of 4 blocks, 32 bytes each):

Block 0          Block 1          Block 2          Block 3
┌────────────┐   ┌────────────┐   ┌────────────┐   ┌────────────┐
│ next ──────┼──►│ next ──────┼──►│ next ──────┼──►│ next: null │
│            │   │            │   │            │   │            │
│  (32 bytes │   │  (32 bytes │   │  (32 bytes │   │  (32 bytes │
│   unused)  │   │   unused)  │   │   unused)  │   │   unused)  │
└────────────┘   └────────────┘   └────────────┘   └────────────┘
↑
m_freeList (head)
```

**Allocation** = pop the head of the free list:

```
alloc():  return Block 0, advance head to Block 1

Block 0          Block 1          Block 2          Block 3
┌────────────┐   ┌────────────┐   ┌────────────┐   ┌────────────┐
│ ██ IN USE █│   │ next ──────┼──►│ next ──────┼──►│ next: null │
│ ██████████ │   │            │   │            │   │            │
└────────────┘   └────────────┘   └────────────┘   └────────────┘
                 ↑
                 m_freeList (new head)
```

**Deallocation** = push the freed block back to the head:

```
dealloc(Block 0):  write "next = old head" into Block 0, make it the new head

Block 0          Block 1          Block 2          Block 3
┌────────────┐   ┌────────────┐   ┌────────────┐   ┌────────────┐
│ next ──────┼──►│ next ──────┼──►│ next ──────┼──►│ next: null │
│            │   │            │   │            │   │            │
└────────────┘   └────────────┘   └────────────┘   └────────────┘
↑
m_freeList (head again)
```

**Both operations are O(1) — just pointer manipulation, no searching.**

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 10.4 The Intrusive Trick — Zero Overhead</h2>

The free list stores its "next" pointer **inside** the free block's own memory. When a block is free, its bytes aren't being used by any live object, so we can treat the first `sizeof(void*)` bytes as a pointer:

```cpp
struct FreeBlock {
    FreeBlock* next;
};
```

A `FreeBlock*` is just a **reinterpret_cast** over the raw memory of the chunk. No extra metadata, no headers, no wasted space — the "bookkeeping" is hidden inside the free blocks' own bytes.

**Requirement:** Each block must be at least `sizeof(void*)` bytes (8 bytes on 64-bit). If your object is smaller (e.g., a `char`), the block size is bumped up to `sizeof(void*)`.

```
Why "intrusive"?

  Normal linked list:    [data] → [Node: data + next ptr] → …   (extra allocation per node)
  Intrusive free list:   [next ptr overlaid on free chunk] → …   (zero extra memory)
```

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 10.5 Alignment in a Pool</h2>

Each block must start at an address that satisfies `alignof(T)`. Since blocks are laid out contiguously, the **block size** must be a multiple of the alignment:

```cpp
// Effective block size: max(sizeof(T), sizeof(void*)), rounded up to alignof(T)
constexpr size_t block_size = align_up(std::max(sizeof(T), sizeof(FreeBlock)), alignof(T));
```

<br>

#### Dissecting the line step by step

This single line does **three things** inside-out:

**Step 1 — `std::max(sizeof(T), sizeof(FreeBlock))`**

Each block serves **two roles** at different times:
- **When allocated**: holds a `T` object → needs `sizeof(T)` bytes
- **When free**: holds a `FreeBlock*` next pointer → needs `sizeof(FreeBlock)` = `sizeof(void*)` = 8 bytes on 64-bit

The block must be large enough for **whichever is bigger**.

**Step 2 — `align_up(value, alignof(T))`**

Blocks sit **back to back** in the buffer. If the block size isn't a multiple of `alignof(T)`, the **second** block starts at a misaligned address. `align_up` rounds up to the **next multiple** of the alignment.

**Step 3 — Result**

The **smallest block size** that (a) fits a `T`, (b) fits a `FreeBlock` pointer, and (c) is a multiple of `T`'s alignment so every block starts at a valid address.

<br>

#### How `align_up` works (the bit trick)

```cpp
std::size_t align_up(std::size_t n, std::size_t alignment) {
    return (n + alignment - 1) & ~(alignment - 1);
}
```

In plain words: "Give me the smallest number ≥ `n` that is a multiple of `alignment`."

Walking through `align_up(10, 4)`:

```
alignment = 4         → binary: 0100
alignment - 1 = 3     → binary: 0011
~(alignment - 1) = ~3 → binary: 1100   (mask: clears the bottom 2 bits)

n = 10                → binary: 1010
n + 3 = 13            → binary: 1101
13 & 1100 = 12        → binary: 1100   ← next multiple of 4  ✓
```

It adds `alignment - 1` (to push past the next boundary if not already on one), then **masks off** the low bits to snap down to the boundary. Only works for power-of-two alignments — which all C++ types have.

<br>

#### Example A: `T` = Widget (24 bytes), `alignof(T) = 8`

```
Step 1: max(24, 8) = 24
Step 2: align_up(24, 8) = 24      ← 24 is already a multiple of 8

Block layout (all blocks start at multiples of 8 ✓):
addr:   0        24       48       72
        ┌────────┬────────┬────────┬────────┐
        │ B0     │ B1     │ B2     │ B3     │
        │ 24 B   │ 24 B   │ 24 B   │ 24 B   │
        └────────┴────────┴────────┴────────┘
        0%8=0 ✓  24%8=0 ✓  48%8=0 ✓  72%8=0 ✓
```

#### Example B: `T` is 10 bytes, `alignof(T) = 4`

```
Step 1: max(10, 8) = 10
Step 2: align_up(10, 4) = 12      ← 10 is NOT a multiple of 4, round up to 12

Without align_up (block_size = 10, WRONG):
addr:   0        10       20       30
        ┌────────┬────────┬────────┬────────┐
        │ B0     │ B1     │ B2     │ B3     │
        └────────┴────────┴────────┴────────┘
        0%4=0 ✓  10%4=2 ✗  20%4=0 ✓  30%4=2 ✗
                      ↑ MISALIGNED!

With align_up (block_size = 12, CORRECT):
addr:   0        12       24       36
        ┌────────┬────────┬────────┬────────┐
        │ B0     │ B1     │ B2     │ B3     │
        │ 12 B   │ 12 B   │ 12 B   │ 12 B   │
        └────────┴────────┴────────┴────────┘
        0%4=0 ✓  12%4=0 ✓  24%4=0 ✓  36%4=0 ✓
                 (2 bytes padding per block — the price of alignment)
```

#### Example C: `T = char` (1 byte), `alignof(char) = 1`

```
Step 1: max(1, 8) = 8        ← must fit a FreeBlock pointer
Step 2: align_up(8, 1) = 8   ← everything is a multiple of 1

block_size = 8 (7 bytes "wasted" per block, but unavoidable)
```

#### Example D: `T` has `alignof(T) = 16` (e.g. SIMD type), `sizeof(T) = 20`

```
Step 1: max(20, 8) = 20
Step 2: align_up(20, 16) = 32    ← next multiple of 16 after 20

block_size = 32 (12 bytes padding per block)
addr:   0        32       64       96
        all start at multiples of 16 ✓
```

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 10.6 Pool vs Arena vs <code>malloc</code> — When to Use What</h2>

| | Arena | Pool | `malloc` / `new` |
|---|-------|------|-------------------|
| Object sizes | Any | **Fixed** (one size per pool) | Any |
| Individual free | No | **Yes** | Yes |
| Alloc speed | Fastest (bump) | Very fast (pop free list) | Slowest (search + bookkeep) |
| Free speed | N/A (bulk `reset()`) | Very fast (push free list) | Slow (coalesce + bookkeep) |
| Fragmentation | None (external) | **None** (fixed-size → no splitting) | Can be severe |
| Cache locality | Excellent | **Very good** (contiguous blocks) | Variable |
| Use case | Same-lifetime batches | Same-size, independent lifetimes | General purpose |

<br><br>

---
---

# PART 3: IMPLEMENTATION

---
---

<br>

<h2 style="color: #2980B9;">📘 10.7 Pool Allocator — Interface</h2>

```cpp
template <typename T>
class PoolAllocator {
public:
    explicit PoolAllocator(std::size_t num_blocks);
    ~PoolAllocator();

    PoolAllocator(const PoolAllocator&) = delete;
    PoolAllocator& operator=(const PoolAllocator&) = delete;

    // Allocate one T-sized block (returns raw memory, no construction)
    void* allocate();

    // Deallocate one block
    void  deallocate(void* ptr);

    // Allocate + construct
    template <typename... Args>
    T* create(Args&&... args);

    // Destruct + deallocate
    void destroy(T* obj);

    // Stats
    std::size_t capacity() const;
    std::size_t used() const;
    std::size_t available() const;
};
```

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 10.8 The Free Block Node</h2>

```cpp
struct FreeBlock {
    FreeBlock* next;
};
```

That's literally it. When a block is free, we overlay this struct on its memory. When the block is allocated to a user, those bytes become the user's object.

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 10.9 Initialization — Building the Free List</h2>

At construction, the pool allocates one big buffer and threads all blocks into a linked list:

```cpp
template <typename T>
PoolAllocator<T>::PoolAllocator(std::size_t num_blocks) {
    static_assert(sizeof(T) >= sizeof(FreeBlock) || sizeof(FreeBlock) <= block_size,
        "block must fit a FreeBlock pointer");

    m_blockSize = align_up(std::max(sizeof(T), sizeof(FreeBlock)), alignof(T));
    m_numBlocks = num_blocks;
    m_buffer = static_cast<char*>(std::aligned_alloc(alignof(T), m_blockSize * num_blocks));

    // Thread all blocks into the free list
    m_freeList = nullptr;
    for (std::size_t i = num_blocks; i > 0; --i) {
        char* blockAddr = m_buffer + (i - 1) * m_blockSize;
        auto* node = reinterpret_cast<FreeBlock*>(blockAddr);
        node->next = m_freeList;
        m_freeList = node;
    }
    m_usedCount = 0;
}
```

Why loop **backwards**? So that `m_freeList` ends up pointing to Block 0, and blocks are allocated in **ascending address order** — better cache locality for early allocations.

```
After init (4 blocks):

m_freeList → [B0] → [B1] → [B2] → [B3] → nullptr
```

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 10.10 Allocate — Pop the Head</h2>

```cpp
void* PoolAllocator<T>::allocate() {
    if (!m_freeList) {
        throw std::bad_alloc();   // pool exhausted
    }
    FreeBlock* block = m_freeList;
    m_freeList = block->next;
    ++m_usedCount;
    return static_cast<void*>(block);
}
```

**One pointer read, one pointer write, one increment.** That's all. Compare with `malloc`'s search-split-bookkeep sequence.

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 10.11 Deallocate — Push to Head</h2>

```cpp
void PoolAllocator<T>::deallocate(void* ptr) {
    auto* block = static_cast<FreeBlock*>(ptr);
    block->next = m_freeList;
    m_freeList = block;
    --m_usedCount;
}
```

The freed block becomes the new head. **O(1), no coalescing, no searching.**

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 10.12 Create & Destroy — Placement new + Explicit Destructor</h2>

```cpp
template <typename T>
template <typename... Args>
T* PoolAllocator<T>::create(Args&&... args) {
    void* mem = allocate();
    return new (mem) T(std::forward<Args>(args)...);   // placement new
}

template <typename T>
void PoolAllocator<T>::destroy(T* obj) {
    obj->~T();            // explicit destructor call
    deallocate(obj);      // return block to free list
}
```

Separation of **allocation** (memory) from **construction** (object) — the same principle as Day 8 placement new.

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 10.13 Destructor — Clean Up</h2>

```cpp
template <typename T>
PoolAllocator<T>::~PoolAllocator() {
    std::free(m_buffer);
}
```

The pool does **not** call destructors for in-use objects. That's the user's responsibility (via `destroy()` before the pool dies). This is the same trade-off as an arena — if you need guaranteed cleanup, track live objects or use RAII wrappers.

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 10.14 Memory Layout Diagram</h2>

```
PoolAllocator<Widget> pool(5);   // Widget is 24 bytes, alignof = 8
block_size = align_up(max(24, 8), 8) = 24

m_buffer (one contiguous allocation):
┌─────────────┬─────────────┬─────────────┬─────────────┬─────────────┐
│  Block 0    │  Block 1    │  Block 2    │  Block 3    │  Block 4    │
│  24 bytes   │  24 bytes   │  24 bytes   │  24 bytes   │  24 bytes   │
│  addr: 0    │  addr: 24   │  addr: 48   │  addr: 72   │  addr: 96   │
└─────────────┴─────────────┴─────────────┴─────────────┴─────────────┘

Free list (all blocks free):
m_freeList → B0 → B1 → B2 → B3 → B4 → nullptr

After create(), create(), create():
m_freeList → B3 → B4 → nullptr

┌─────────────┬─────────────┬─────────────┬─────────────┬─────────────┐
│ ████ USED ██│ ████ USED ██│ ████ USED ██│  [next→B4]  │  [next→nil] │
│  Widget A   │  Widget B   │  Widget C   │   (free)    │   (free)    │
└─────────────┴─────────────┴─────────────┴─────────────┴─────────────┘

After destroy(B):   (Widget B freed — pushed to head)
m_freeList → B1 → B3 → B4 → nullptr

┌─────────────┬─────────────┬─────────────┬─────────────┬─────────────┐
│ ████ USED ██│  [next→B3]  │ ████ USED ██│  [next→B4]  │  [next→nil] │
│  Widget A   │   (free)    │  Widget C   │   (free)    │   (free)    │
└─────────────┴─────────────┴─────────────┴─────────────┴─────────────┘
```

Notice: after returning Block 1, the free list is **not** in address order (B1 → B3 → B4). That's fine — order doesn't matter for correctness. Over time, the free list becomes "shuffled." This can hurt cache locality slightly compared to sequential arena allocation, but each alloc/dealloc stays O(1).

<br><br>

---
---

# PART 4: PERFORMANCE — WHY POOLS WIN

---
---

<br>

<h2 style="color: #2980B9;">📘 10.15 Benchmark: Pool vs <code>new/delete</code></h2>

Typical results for 1 million alloc/dealloc cycles of a 64-byte object:

```
allocator         alloc+dealloc time
──────────────    ──────────────────
new/delete        ~180,000 µs
pool              ~3,000 µs          (~60x faster)
arena (batch)     ~1,200 µs          (fastest, but no individual free)
```

Why pools win:
1. **No searching** — alloc is one pointer pop
2. **No coalescing** — dealloc is one pointer push
3. **No metadata headers** — intrusive list uses block memory itself
4. **Cache friendly** — all blocks live in one contiguous buffer
5. **No system calls** — memory is pre-allocated

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 10.16 Cache Locality Deep Dive</h2>

When you call `new` repeatedly, the default allocator may scatter objects across the heap. A pool keeps them **contiguous**:

```
Default allocator (scattered):
  Widget A @ 0x7f0012    Widget B @ 0x7f8800    Widget C @ 0x7fA040
  → iterating over A, B, C causes cache misses (different cache lines, maybe different pages)

Pool allocator (contiguous):
  Widget A @ 0x1000    Widget B @ 0x1040    Widget C @ 0x1080
  → iterating over A, B, C: all in nearby cache lines, prefetcher works well
```

This matters enormously for **entity systems**, **particle systems**, and any pattern where you iterate over many same-type objects every frame.

<br><br>

---
---

# PART 5: EDGE CASES & PITFALLS

---
---

<br>

<h2 style="color: #2980B9;">📘 10.17 Common Mistakes</h2>

<br>

#### Pitfall 1: Double free

```cpp
pool.destroy(obj);
pool.destroy(obj);   // UB: block was already pushed to free list
                     // now free list has a cycle → infinite loop or corruption
```

A debug build might stamp freed blocks with a sentinel (e.g. `0xDEADBEEF`) and check on `deallocate`.

<br>

#### Pitfall 2: Freeing a pointer not from this pool

```cpp
int* foreign = new int(42);
pool.deallocate(foreign);   // UB: foreign is not inside m_buffer
```

Add a bounds check in debug mode:

```cpp
void deallocate(void* ptr) {
    assert(ptr >= m_buffer && ptr < m_buffer + m_blockSize * m_numBlocks);
    // ...
}
```

<br>

#### Pitfall 3: Forgetting to call destructors

```cpp
{
    PoolAllocator<std::string> pool(100);
    std::string* s = pool.create("hello world");
    // pool goes out of scope — ~PoolAllocator frees buffer
    // but ~string() was never called → leaked dynamic string buffer
}
```

Always pair `create()` with `destroy()`, or track live objects.

<br>

#### Pitfall 4: Using a block after destroy

```cpp
Widget* w = pool.create(42);
pool.destroy(w);
w->doSomething();   // UB: w's memory now contains a FreeBlock* next pointer
```

After `destroy`, the block's bytes are overwritten with the intrusive `next` pointer. Any use is undefined.

<br><br>

---
---

# PART 6: REAL-WORLD USAGE

---
---

<br>

<h2 style="color: #2980B9;">📘 10.18 Where Pool Allocators Appear</h2>

| Domain | Usage |
|--------|-------|
| **Game engines** (Unreal, Unity internals) | Entity Component Systems — each component type gets its own pool |
| **Networking** | Fixed-size packet buffers, connection state objects |
| **Compilers** | AST nodes (often all the same size or a few sizes) |
| **Database engines** | Buffer page pools, row slot pools |
| **OS kernels** | Linux **slab allocator** (`kmem_cache`) — a production-grade pool per kernel object type |
| **Embedded** | Memory-constrained systems where `malloc` overhead is unacceptable |

<br>

#### Linux slab allocator (bonus context)

The Linux kernel's slab allocator is essentially a pool allocator per object type:

```
kmem_cache_create("task_struct", sizeof(task_struct), ...);
// Creates a pool of task_struct-sized blocks
// alloc: kmem_cache_alloc()   → pop from free list
// free:  kmem_cache_free()    → push to free list
```

It adds slabs (pages of blocks), per-CPU caches, and NUMA awareness — but the core is the same free-list pool from this lesson.

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 10.19 Multi-Size Pools</h2>

Real allocators often keep **multiple pools** for different size classes:

```
Size class pools:
  Pool-8:    blocks of 8 bytes    (for objects ≤ 8)
  Pool-16:   blocks of 16 bytes   (for objects 9-16)
  Pool-32:   blocks of 32 bytes   (for objects 17-32)
  Pool-64:   blocks of 64 bytes   (for objects 33-64)
  Pool-128:  blocks of 128 bytes  (for objects 65-128)
  …
  Fallback:  malloc for anything larger
```

`jemalloc` and `tcmalloc` work this way internally — they're essentially arrays of pool allocators for common sizes, with thread-local caches.

<br><br>

---
---

# PART 7: BUILD EXERCISE

---
---

<br>

<h2 style="color: #2980B9;">📘 10.20 Exercise: Build a Pool Allocator</h2>

Implement a complete `PoolAllocator<T>` with the following features:

```cpp
#include <cstddef>
#include <cstdlib>
#include <cassert>
#include <new>
#include <utility>
#include <iostream>
#include <algorithm>

inline std::size_t align_up(std::size_t n, std::size_t alignment) {
    return (n + alignment - 1) & ~(alignment - 1);
}

struct FreeBlock {
    FreeBlock* next;
};

template <typename T>
class PoolAllocator {
private:
    char*       m_buffer    = nullptr;
    FreeBlock*  m_freeList  = nullptr;
    std::size_t m_blockSize = 0;
    std::size_t m_numBlocks = 0;
    std::size_t m_usedCount = 0;

public:
    explicit PoolAllocator(std::size_t num_blocks)
        : m_numBlocks(num_blocks)
    {
        m_blockSize = align_up(std::max(sizeof(T), sizeof(FreeBlock)), alignof(T));
        // YOUR IMPLEMENTATION:
        // 1. Allocate m_buffer with aligned_alloc (or malloc if alignment ≤ max_align_t)
        // 2. Thread all blocks into the free list (loop backwards for ascending order)
    }

    ~PoolAllocator() {
        // YOUR IMPLEMENTATION: free m_buffer
    }

    PoolAllocator(const PoolAllocator&) = delete;
    PoolAllocator& operator=(const PoolAllocator&) = delete;

    void* allocate() {
        // YOUR IMPLEMENTATION: pop head of free list, throw bad_alloc if empty
    }

    void deallocate(void* ptr) {
        // YOUR IMPLEMENTATION: push ptr to head of free list
        // BONUS: assert ptr is within m_buffer range
    }

    template <typename... Args>
    T* create(Args&&... args) {
        // YOUR IMPLEMENTATION: allocate() + placement new
    }

    void destroy(T* obj) {
        // YOUR IMPLEMENTATION: call destructor + deallocate()
    }

    std::size_t capacity()  const { return m_numBlocks; }
    std::size_t used()      const { return m_usedCount; }
    std::size_t available() const { return m_numBlocks - m_usedCount; }
};
```

<br>

### Test Driver

```cpp
struct Widget {
    int id;
    Widget(int i) : id(i) { std::cout << "  Widget(" << id << ") constructed\n"; }
    ~Widget() { std::cout << "  Widget(" << id << ") destroyed\n"; }
    void greet() const { std::cout << "  Hello from Widget " << id << "\n"; }
};

int main() {
    std::cout << "=== 1. Basic create/destroy ===\n";
    {
        PoolAllocator<Widget> pool(4);
        std::cout << "  capacity=" << pool.capacity()
                  << " used=" << pool.used()
                  << " available=" << pool.available() << "\n";

        Widget* a = pool.create(1);
        Widget* b = pool.create(2);
        Widget* c = pool.create(3);
        a->greet();
        b->greet();
        c->greet();

        std::cout << "  used=" << pool.used() << "\n";

        pool.destroy(b);
        std::cout << "  after destroying Widget 2: used=" << pool.used() << "\n";

        Widget* d = pool.create(4);
        d->greet();
        std::cout << "  used=" << pool.used() << "\n";

        pool.destroy(a);
        pool.destroy(c);
        pool.destroy(d);
    }

    std::cout << "\n=== 2. Pool exhaustion ===\n";
    {
        PoolAllocator<int> pool(2);
        int* a = pool.create(10);
        int* b = pool.create(20);
        try {
            int* c = pool.create(30);  // should throw
            (void)c;
        } catch (const std::bad_alloc& e) {
            std::cout << "  caught bad_alloc: pool full!\n";
        }
        pool.destroy(a);
        pool.destroy(b);
    }

    std::cout << "\n=== 3. Reuse after free ===\n";
    {
        PoolAllocator<Widget> pool(2);
        Widget* a = pool.create(100);
        pool.destroy(a);

        Widget* b = pool.create(200);
        b->greet();
        std::cout << "  same address reused: " << (a == b ? "yes" : "no") << "\n";
        pool.destroy(b);
    }

    std::cout << "\n=== 4. Benchmark: pool vs new/delete ===\n";
    {
        constexpr int N = 1'000'000;

        PoolAllocator<int> pool(1);

        auto start = std::chrono::high_resolution_clock::now();
        for (int i = 0; i < N; ++i) {
            int* p = pool.create(i);
            pool.destroy(p);
        }
        auto pool_time = std::chrono::high_resolution_clock::now() - start;

        start = std::chrono::high_resolution_clock::now();
        for (int i = 0; i < N; ++i) {
            int* p = new int(i);
            delete p;
        }
        auto new_time = std::chrono::high_resolution_clock::now() - start;

        std::printf("  Pool:       %lld us\n",
            std::chrono::duration_cast<std::chrono::microseconds>(pool_time).count());
        std::printf("  new/delete: %lld us\n",
            std::chrono::duration_cast<std::chrono::microseconds>(new_time).count());
    }

    std::cout << "\n=== Done ===\n";
    return 0;
}
```

<br>

### Compile and run

```bash
g++ -std=c++17 -O2 -Wall -Wextra -fsanitize=address -o day10 day10_pool.cpp && ./day10
```

<br><br>

---

<br>

#### Bonus Challenges

1. **Debug sentinel**: On `deallocate`, stamp the first 4 bytes of the block with `0xDEADBEEF`. On `allocate`, assert the sentinel is present (catches double-free and use-after-free).

2. **Growable pool**: When all blocks are used, allocate a new slab (another contiguous buffer) and chain it. Track slabs in a `std::vector<char*>` for cleanup.

3. **Thread-safe pool**: Use `std::atomic<FreeBlock*>` for `m_freeList` with compare-and-swap. This is a lock-free stack — the classic Treiber stack.

4. **Multiple size classes**: Build a `MultiPool` that owns several `PoolAllocator`s for sizes 8, 16, 32, 64, 128, and routes `allocate(size)` to the right pool.

<br><br>

---
---

# PART 8: DEEP DIVE — COMMON QUESTIONS

---
---

<br>

<h2 style="color: #2980B9;">📘 10.21 Q&A</h2>

<br>

#### Q1: "Why store the free list pointer *inside* the free block instead of a separate data structure?"

Storing "next" pointers separately (e.g. a `std::vector<void*>`) wastes memory and is slower (extra cache line for the bookkeeping structure). The intrusive approach uses **zero additional memory** because free blocks aren't holding user data anyway. It's the same principle behind intrusive linked lists in the Linux kernel.

<br>

#### Q2: "What if my object is smaller than a pointer?"

The block size is forced to `max(sizeof(T), sizeof(FreeBlock))`. A `char` pool with 8-byte blocks wastes 7 bytes per block, but that's unavoidable — you need room for the `next` pointer when the block is free. In practice, pools are used for objects ≥ 16-32 bytes where this overhead is negligible.

<br>

#### Q3: "Can I use a pool allocator with STL containers?"

Yes — Day 11 covers the STL allocator interface. You'd write a thin wrapper conforming to `std::allocator_traits`. The pool handles `allocate()`/`deallocate()`, and the container handles construction/destruction via `allocator_traits::construct`/`destroy`.

<br>

#### Q4: "How does the free list order affect performance?"

After many alloc/dealloc cycles, the free list becomes "shuffled" — blocks aren't in address order. This can hurt iteration cache locality. Some implementations periodically **sort** the free list by address, or use per-cacheline sublists. For most use cases, the shuffling is not a problem because the blocks are all in the same contiguous buffer anyway.

<br>

#### Q5: "Pool vs arena — can I combine them?"

Yes. A common pattern:
- **Arena** for the pool's backing memory (instead of `malloc`)
- **Pool** manages fixed-size blocks within the arena
- **Arena reset** reclaims everything (including all pool blocks)

This is especially useful in game engines: the frame arena owns the pool, and `arena.reset()` at end-of-frame implicitly frees all pooled objects.

<br>

#### Q6: "How does this relate to `std::pmr::pool_resource`?"

C++17 added `std::pmr::synchronized_pool_resource` and `std::pmr::unsynchronized_pool_resource` — standard pool allocators that work with the `std::pmr` (polymorphic memory resource) framework. They use the same free-list concept internally but support multiple size classes and integrate with `std::pmr::vector`, `std::pmr::string`, etc.

<br><br>

---

## Reflection Questions

1. When would a pool allocator outperform an arena? When would an arena be better?
2. Why is the intrusive free list considered "zero overhead"?
3. What happens to cache locality as blocks are allocated and freed in random order?
4. How would you detect a double-free bug in a pool allocator?
5. Why does the Linux kernel use slab allocation for kernel objects?

---

## Interview Questions

1. "What is a pool allocator and when would you use one?"
2. "Explain how an intrusive free list works."
3. "What's the time complexity of alloc and dealloc in a pool?"
4. "How do you handle alignment in a pool allocator?"
5. "Compare pool allocator vs arena allocator — trade-offs?"
6. "How would you make a pool allocator thread-safe?"
7. "What is the slab allocator in the Linux kernel?"
8. "How would you detect use-after-free in a pool allocator?"

---

**Next**: Day 11 — STL Allocator Interface →
