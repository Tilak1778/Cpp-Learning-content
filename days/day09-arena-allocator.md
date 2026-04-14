# Day 9: Arena (Bump) Allocator

[← Back to Study Plan](../lld-study-plan.md) | [← Day 8](day08-new-delete-placement-new.md)

> **Time**: ~1.5-2 hours
> **Goal**: Build an arena allocator from scratch — the simplest and fastest allocator possible. Understand when linear allocation crushes `malloc`, and why "no individual free" is a feature, not a limitation.

---
---

# PART 1: THE PROBLEM WITH `malloc` / `new`

---
---

<br>

<h2 style="color: #2980B9;">📘 9.1 Why General-Purpose Allocators Are Slow</h2>

`malloc` is a **general-purpose** allocator. It has to handle every possible allocation pattern — any size, any order of free, any lifetime. This generality costs performance:

```
What malloc must do on every allocation:
  1. Search free lists for a fitting block (first-fit, best-fit, etc.)
  2. Split the block if it's too large
  3. Update bookkeeping (linked lists, headers, footers)
  4. Handle thread safety (locks or thread-local caches)

What malloc must do on every free:
  1. Mark the block as free
  2. Coalesce with adjacent free blocks (to prevent fragmentation)
  3. Update bookkeeping
  4. Possibly return memory to the OS (munmap)

Each malloc/free: ~50-200 nanoseconds (varies by implementation)
```

For many real-world workloads, this is overkill. If you know your allocation pattern in advance, you can build something **10-100x faster**.

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 9.2 Common Allocation Patterns Where malloc Is Wasteful</h2>

| Pattern | Description | Example |
|---------|-------------|---------|
| **Frame-based** | Allocate a bunch, free ALL at once | Game frame: allocate particles, render, reset |
| **Phase-based** | Allocate during a phase, discard everything at phase end | Compiler: parse phase allocations, discard after AST built |
| **Request-scoped** | Allocate per request, free at request end | HTTP server: build response, send, discard |
| **Same-size objects** | Many objects of identical size | Network packets, AST nodes, entity components |

For all these patterns, you don't need per-object free. You need:
- **Fast allocation** (just bump a pointer)
- **Bulk deallocation** (reset everything at once)

That's exactly what an arena does.

<br><br>

---
---

# PART 2: HOW AN ARENA ALLOCATOR WORKS

---
---

<br>

<h2 style="color: #2980B9;">📘 9.3 The Core Idea — Bump the Pointer</h2>

An arena pre-allocates a **large contiguous buffer** and hands out memory by moving a pointer forward:

```
Initial state:
┌──────────────────────────────────────────────────────┐
│                   Arena Buffer (e.g., 1 MB)          │
└──────────────────────────────────────────────────────┘
↑
m_offset = 0

After alloc(32):
┌──────────┬───────────────────────────────────────────┐
│ 32 bytes │               free space                  │
│ (used)   │                                           │
└──────────┴───────────────────────────────────────────┘
            ↑
            m_offset = 32

After alloc(64):
┌──────────┬───────────────┬───────────────────────────┐
│ 32 bytes │   64 bytes    │         free space         │
│          │               │                            │
└──────────┴───────────────┴───────────────────────────┘
                            ↑
                            m_offset = 96

After alloc(16):
┌──────────┬───────────────┬────────┬──────────────────┐
│ 32 bytes │   64 bytes    │16 bytes│    free space     │
└──────────┴───────────────┴────────┴──────────────────┘
                                     ↑
                                     m_offset = 112

After reset():
┌──────────────────────────────────────────────────────┐
│                   Arena Buffer (all free again)       │
└──────────────────────────────────────────────────────┘
↑
m_offset = 0   (everything "freed" in one instruction!)
```

**Allocation = increment a pointer. That's it.** No searching, no bookkeeping, no fragmentation.

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 9.4 Handling Alignment</h2>

Types have alignment requirements (Day 8). The arena must **pad the offset** to satisfy alignment before each allocation:

```
Current state: m_offset = 5
Request: alloc<int>() → needs alignof(int) = 4

  offset 5 is NOT a multiple of 4
  → round up to next multiple of 4 = 8
  → 3 bytes of padding wasted

┌─────┬─────┬──────────┬────────────────────────┐
│used │ pad │ int (4B) │       free space        │
│ 5B  │ 3B  │          │                         │
└─────┴─────┴──────────┴────────────────────────┘
 0    5    8          12
               ↑
               aligned offset
```

The alignment formula:

```cpp
// Round 'offset' up to the next multiple of 'alignment'
std::size_t align_up(std::size_t offset, std::size_t alignment) {
    return (offset + alignment - 1) & ~(alignment - 1);
}

// How it works (alignment must be power of 2):
//   alignment = 8       → binary: 00001000
//   alignment - 1 = 7   → binary: 00000111
//   ~(alignment - 1)    → binary: 11111000  (mask: clears lower bits)
//
//   offset = 13          → binary: 00001101
//   offset + 7 = 20      → binary: 00010100
//   20 & 11111000 = 16   → binary: 00010000  ✓ (next multiple of 8)
```

This bit trick only works when alignment is a power of 2 — which it always is for C++ types.

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 9.5 No Individual Free — Why That's OK</h2>

An arena does **not** support freeing individual allocations. You can only `reset()` the entire arena.

This seems like a huge limitation, but in practice it's perfect for many patterns:

```
Game loop (frame allocator):
  while (running) {
      arena.reset();                          // free everything from last frame
      
      auto* particles = arena.alloc<Particle>(1000);
      auto* commands  = arena.alloc<DrawCommand>(200);
      auto* strings   = arena.alloc<char>(4096);
      
      simulate(particles);
      render(commands);
      // End of frame — all allocations become invalid
  }

Compiler (phase allocator):
  Arena parse_arena(1 * MB);
  AST* ast = parse(source_code, parse_arena);   // all nodes from arena
  
  Arena codegen_arena(2 * MB);
  generate_code(ast, codegen_arena);
  
  parse_arena.reset();     // AST no longer needed
  codegen_arena.reset();   // codegen temps no longer needed
```

The tradeoff:
- **Advantage**: No per-object overhead, no fragmentation, O(1) allocation, O(1) mass deallocation
- **Limitation**: All objects in the arena must share the same lifetime

<br><br>

---
---

# PART 3: IMPLEMENTATION

---
---

<br>

<h2 style="color: #2980B9;">📘 9.6 Basic Arena — Interface</h2>

```cpp
class Arena {
public:
    explicit Arena(std::size_t size_bytes);
    ~Arena();

    // Non-copyable, non-movable (owns raw memory)
    Arena(const Arena&) = delete;
    Arena& operator=(const Arena&) = delete;

    // Allocate raw bytes with specified alignment
    void* alloc(std::size_t size, std::size_t alignment);

    // Allocate and construct a T
    template<typename T, typename... Args>
    T* create(Args&&... args);

    // Allocate an array of T (default-constructed)
    template<typename T>
    T* alloc_array(std::size_t count);

    // Reset — "frees" all allocations (no destructors called!)
    void reset();

    // Stats
    std::size_t used() const;
    std::size_t capacity() const;

private:
    char* m_buffer;          // the backing memory
    std::size_t m_capacity;  // total size
    std::size_t m_offset;    // current allocation point
};
```

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 9.7 Implementation — Step by Step</h2>

```cpp
#include <cstddef>
#include <cstdlib>
#include <cstring>
#include <cassert>
#include <new>
#include <utility>

class Arena {
public:
    explicit Arena(std::size_t size_bytes)
        : m_buffer(static_cast<char*>(std::malloc(size_bytes)))
        , m_capacity(size_bytes)
        , m_offset(0)
    {
        if (!m_buffer) throw std::bad_alloc();
    }

    ~Arena() {
        std::free(m_buffer);
    }

    Arena(const Arena&) = delete;
    Arena& operator=(const Arena&) = delete;

    void* alloc(std::size_t size, std::size_t alignment) {
        // Round offset up to required alignment
        std::size_t aligned_offset = align_up(m_offset, alignment);

        // Check if we have enough space
        if (aligned_offset + size > m_capacity) {
            return nullptr;  // out of memory
            // Alternative: throw, or grow (see 9.9)
        }

        void* ptr = m_buffer + aligned_offset;
        m_offset = aligned_offset + size;
        return ptr;
    }

    template<typename T, typename... Args>
    T* create(Args&&... args) {
        void* mem = alloc(sizeof(T), alignof(T));
        if (!mem) return nullptr;
        return new (mem) T(std::forward<Args>(args)...);  // placement new
    }

    template<typename T>
    T* alloc_array(std::size_t count) {
        void* mem = alloc(sizeof(T) * count, alignof(T));
        if (!mem) return nullptr;

        T* arr = static_cast<T*>(mem);
        for (std::size_t i = 0; i < count; ++i) {
            new (&arr[i]) T();  // default construct each element
        }
        return arr;
    }

    void reset() {
        m_offset = 0;
        // Note: NO destructors called! Only safe for trivial types
        // or when you've manually destructed objects.
    }

    std::size_t used() const { return m_offset; }
    std::size_t capacity() const { return m_capacity; }

private:
    char* m_buffer;
    std::size_t m_capacity;
    std::size_t m_offset;

    static std::size_t align_up(std::size_t offset, std::size_t alignment) {
        return (offset + alignment - 1) & ~(alignment - 1);
    }
};
```

Let's trace through an example:

```
Arena arena(1024);   // 1 KB buffer

int* a = arena.create<int>(42);
//   offset = 0, alignof(int) = 4 → aligned_offset = 0
//   placement new: constructs 42 at buffer[0..3]
//   m_offset = 4

double* b = arena.create<double>(3.14);
//   offset = 4, alignof(double) = 8 → aligned_offset = 8 (4 bytes padding)
//   placement new: constructs 3.14 at buffer[8..15]
//   m_offset = 16

char* name = arena.alloc_array<char>(6);
//   offset = 16, alignof(char) = 1 → aligned_offset = 16 (no padding)
//   constructs 6 zero chars at buffer[16..21]
//   m_offset = 22

Memory layout:
┌──────┬──────┬────────────────┬────────────┬──────────────┐
│ int  │ pad  │    double      │ char[6]    │   free ...   │
│ 42   │ (4B) │    3.14        │ '\0'×6     │              │
└──────┴──────┴────────────────┴────────────┴──────────────┘
 0     4     8               16           22             1024

arena.reset();
//   m_offset = 0 → entire buffer available again
```

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 9.8 The Destructor Problem</h2>

`reset()` just moves the pointer back — it does NOT call destructors. This is fine for trivially destructible types (int, float, POD structs), but dangerous for types that own resources:

```cpp
Arena arena(4096);

// SAFE — int is trivially destructible
int* x = arena.create<int>(42);
arena.reset();  // fine, nothing to clean up

// DANGEROUS — std::string has a non-trivial destructor
std::string* s = arena.create<std::string>("hello world");
arena.reset();  // LEAK! string's internal buffer never freed!
```

**Solutions:**

<br>

#### Option A: Only use arenas for trivial types

```cpp
template<typename T, typename... Args>
T* create(Args&&... args) {
    static_assert(std::is_trivially_destructible_v<T>,
                  "Arena::create only supports trivially destructible types. "
                  "For non-trivial types, use create_tracked().");
    void* mem = alloc(sizeof(T), alignof(T));
    if (!mem) return nullptr;
    return new (mem) T(std::forward<Args>(args)...);
}
```

<br>

#### Option B: Track non-trivial objects for destruction

Store a list of destructor callbacks and call them on `reset()`:

```cpp
class Arena {
    // ...
    struct DtorEntry {
        void* object;
        void (*destroy)(void*);
    };

    DtorEntry m_dtors[1024];  // fixed-size array (no malloc!)
    std::size_t m_dtor_count = 0;

public:
    template<typename T, typename... Args>
    T* create_tracked(Args&&... args) {
        void* mem = alloc(sizeof(T), alignof(T));
        if (!mem) return nullptr;
        T* obj = new (mem) T(std::forward<Args>(args)...);

        if constexpr (!std::is_trivially_destructible_v<T>) {
            assert(m_dtor_count < 1024);
            m_dtors[m_dtor_count++] = {
                obj,
                [](void* p) { static_cast<T*>(p)->~T(); }
            };
        }
        return obj;
    }

    void reset() {
        // Call destructors in REVERSE order (LIFO — like stack unwinding)
        for (std::size_t i = m_dtor_count; i > 0; --i) {
            m_dtors[i - 1].destroy(m_dtors[i - 1].object);
        }
        m_dtor_count = 0;
        m_offset = 0;
    }
};
```

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 9.9 Growing Arenas — Handling Overflow</h2>

A fixed-size arena is limiting. Real-world arenas typically support growing by chaining multiple buffers:

```
Initial arena (1 KB):
┌────────────────────────────────┐
│ Block 0 (1 KB)                 │  ← m_current_block
└────────────────────────────────┘

After block 0 is full, alloc() creates a new block:
┌────────────────────────────────┐   ┌────────────────────────────────┐
│ Block 0 (1 KB) — FULL          │ → │ Block 1 (2 KB) — current       │
└────────────────────────────────┘   └────────────────────────────────┘

After block 1 is full:
┌──────────┐   ┌──────────────┐   ┌──────────────────────────┐
│ Block 0  │ → │ Block 1      │ → │ Block 2 (4 KB) — current  │
│ (1 KB)   │   │ (2 KB)       │   │                           │
└──────────┘   └──────────────┘   └──────────────────────────┘

reset() frees blocks 1, 2; resets block 0's offset to 0.
(Or: keeps all blocks for reuse.)
```

```cpp
class GrowableArena {
    struct Block {
        char* data;
        std::size_t capacity;
        std::size_t offset;
        Block* next;
    };

    Block* m_head;           // first block (kept after reset)
    Block* m_current;        // block we're allocating from
    std::size_t m_block_size; // doubles each time (or stays fixed)

public:
    explicit GrowableArena(std::size_t initial_size = 4096)
        : m_block_size(initial_size)
    {
        m_head = allocate_block(initial_size);
        m_current = m_head;
    }

    void* alloc(std::size_t size, std::size_t alignment) {
        std::size_t aligned = align_up(m_current->offset, alignment);

        if (aligned + size > m_current->capacity) {
            // Current block is full — allocate a new, larger block
            std::size_t new_size = std::max(m_block_size * 2, aligned + size);
            Block* block = allocate_block(new_size);
            m_current->next = block;
            m_current = block;
            m_block_size = new_size;

            aligned = align_up(m_current->offset, alignment);
        }

        void* ptr = m_current->data + aligned;
        m_current->offset = aligned + size;
        return ptr;
    }

    void reset() {
        // Free all blocks except the first
        Block* block = m_head->next;
        while (block) {
            Block* next = block->next;
            std::free(block->data);
            std::free(block);
            block = next;
        }
        m_head->next = nullptr;
        m_head->offset = 0;
        m_current = m_head;
    }

    ~GrowableArena() {
        Block* block = m_head;
        while (block) {
            Block* next = block->next;
            std::free(block->data);
            std::free(block);
            block = next;
        }
    }

private:
    Block* allocate_block(std::size_t size) {
        Block* b = static_cast<Block*>(std::malloc(sizeof(Block)));
        b->data = static_cast<char*>(std::malloc(size));
        b->capacity = size;
        b->offset = 0;
        b->next = nullptr;
        if (!b->data) throw std::bad_alloc();
        return b;
    }

    static std::size_t align_up(std::size_t off, std::size_t align) {
        return (off + align - 1) & ~(align - 1);
    }
};
```

<br><br>

---
---

# PART 4: PERFORMANCE — WHY ARENAS WIN

---
---

<br>

<h2 style="color: #2980B9;">📘 9.10 Arena vs malloc — Performance Comparison</h2>

```
Allocating 1,000,000 small objects (32 bytes each):

malloc/new:
  - Each call: search free list, split block, update metadata
  - ~100-200 ns per allocation
  - Total: ~100-200 ms
  - Freeing: ~100-200 ms (coalescing, etc.)

Arena:
  - Each call: add size to offset (one integer addition)
  - ~2-5 ns per allocation
  - Total: ~2-5 ms
  - Freeing: set offset = 0 (one instruction)
  - Total free: ~0 ns

Speedup: 30-100x faster
```

Why is the arena so fast?

| Property | Arena | malloc |
|----------|-------|--------|
| **Allocation** | Pointer bump (1 add + 1 compare) | Free list search + split + bookkeeping |
| **Deallocation** | Single pointer reset | Per-object free + coalesce + bookkeeping |
| **Cache locality** | Excellent — sequential in memory | Poor — scattered across heap |
| **Fragmentation** | Zero (until reset) | Internal + external fragmentation |
| **Thread safety** | None needed (one owner) or simple atomic bump | Complex (thread-local caches, locks) |
| **Metadata overhead** | Zero per allocation | 8-16 bytes per allocation (headers) |

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 9.11 Cache Locality — The Hidden Advantage</h2>

Arena allocations are **contiguous in memory**. This is a huge win for cache performance:

```
Arena allocations:
┌────┬────┬────┬────┬────┬────┬────┬────┐
│ A  │ B  │ C  │ D  │ E  │ F  │ G  │ H  │  ← all on same cache line(s)
└────┴────┴────┴────┴────┴────┴────┴────┘
  Iterating A-H: all in L1 cache → fast!

malloc allocations (over time, after many alloc/free cycles):
┌────┐   ┌────┐         ┌────┐      ┌────┐
│ A  │   │ C  │         │ E  │      │ G  │  ← scattered across heap
└────┘   └────┘         └────┘      └────┘
     ┌────┐   ┌────┐         ┌────┐      ┌────┐
     │ B  │   │ D  │         │ F  │      │ H  │
     └────┘   └────┘         └────┘      └────┘
  Iterating A-H: cache misses on nearly every access → slow!
```

For workloads that iterate over many small objects (particles, entities, AST nodes), arena allocation can give 5-10x speedups purely from better cache utilization.

<br><br>

---
---

# PART 5: REAL-WORLD USAGE

---
---

<br>

<h2 style="color: #2980B9;">📘 9.12 Where Arenas Are Used in Practice</h2>

| System | How It Uses Arenas |
|--------|-------------------|
| **Game engines** (Unreal, Unity) | Per-frame allocator — all temporary allocations reset every frame (~16ms) |
| **Compilers** (Clang/LLVM, GCC) | Per-phase arena — AST nodes, type info allocated from arena, freed after code generation |
| **Browsers** (Chrome, Firefox) | Per-document arena — DOM nodes allocated from arena, freed when tab closes |
| **Databases** (PostgreSQL) | Per-query memory context — query plan nodes allocated, freed after query completes |
| **Protobuf** | `google::protobuf::Arena` — allocate message objects, free all at once |

<br>

#### Google Protobuf Arena Example

```cpp
#include <google/protobuf/arena.h>

google::protobuf::Arena arena;

// Allocate protobuf messages on the arena
MyMessage* msg = google::protobuf::Arena::CreateMessage<MyMessage>(&arena);
msg->set_name("hello");

// Sub-messages also allocated on the same arena
SubMessage* sub = msg->mutable_sub();

// When 'arena' goes out of scope → all messages freed at once
// No individual delete needed
```

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 9.13 Temporary Allocator / Scratch Buffer Pattern</h2>

A common pattern: use a **thread-local** arena as a scratch buffer for temporary allocations:

```cpp
thread_local Arena t_scratch(64 * 1024);  // 64 KB per thread

std::string process_request(const Request& req) {
    // Save current position (to restore later)
    auto saved_offset = t_scratch.used();

    // All temporary allocations come from the scratch arena
    char* temp_buf = t_scratch.alloc_array<char>(req.size());
    int* indices   = t_scratch.alloc_array<int>(req.count());

    // ... do work using temp_buf and indices ...

    std::string result = build_result(temp_buf, indices);

    // "Free" temporaries by rewinding to saved position
    t_scratch.rewind(saved_offset);

    return result;
}
```

This is essentially a manual, scoped stack allocation — faster than `alloca` (which can't handle alignment properly) and safer (fixed maximum size, no stack overflow risk).

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 9.14 Arena + RAII: <code>ArenaScope</code></h2>

Combine the scratch pattern with RAII for automatic rewind:

```cpp
class ArenaScope {
    Arena& m_arena;
    std::size_t m_saved_offset;
public:
    explicit ArenaScope(Arena& arena)
        : m_arena(arena)
        , m_saved_offset(arena.used())
    {}

    ~ArenaScope() {
        m_arena.rewind(m_saved_offset);
    }

    ArenaScope(const ArenaScope&) = delete;
    ArenaScope& operator=(const ArenaScope&) = delete;
};

// Usage:
void do_work(Arena& arena) {
    ArenaScope scope(arena);  // saves position

    int* data = arena.alloc_array<int>(1000);
    // ... work with data ...

}  // scope destructor rewinds the arena — data is "freed"
```

You'll need to add a `rewind()` method to your Arena:

```cpp
void rewind(std::size_t saved_offset) {
    assert(saved_offset <= m_offset);
    m_offset = saved_offset;
}
```

<br><br>

---
---

# PART 6: BUILD EXERCISE

---
---

<br>

<h2 style="color: #2980B9;">📘 9.15 Exercise: Build an Arena Allocator</h2>

Implement a complete arena allocator with the following API:

```cpp
class Arena {
public:
    explicit Arena(std::size_t capacity);
    ~Arena();

    Arena(const Arena&) = delete;
    Arena& operator=(const Arena&) = delete;

    // Raw allocation with alignment
    void* alloc(std::size_t size, std::size_t alignment = alignof(std::max_align_t));

    // Typed allocation with construction
    template<typename T, typename... Args>
    T* create(Args&&... args);

    // Array allocation with default construction
    template<typename T>
    T* alloc_array(std::size_t count);

    // Free all allocations
    void reset();

    // Rewind to a saved position
    void rewind(std::size_t saved_offset);

    // Query
    std::size_t used() const;
    std::size_t capacity() const;
    std::size_t remaining() const;
};

class ArenaScope {
public:
    explicit ArenaScope(Arena& arena);
    ~ArenaScope();
    ArenaScope(const ArenaScope&) = delete;
    ArenaScope& operator=(const ArenaScope&) = delete;
};
```

<br>

#### Test Driver

```cpp
int main() {
    Arena arena(4096);

    // Basic allocations
    int* a = arena.create<int>(42);
    double* b = arena.create<double>(3.14);
    assert(*a == 42);
    assert(*b == 3.14);

    std::printf("Used: %zu / %zu bytes\n", arena.used(), arena.capacity());

    // Array allocation
    int* arr = arena.alloc_array<int>(100);
    for (int i = 0; i < 100; ++i) arr[i] = i;

    std::printf("After array: %zu / %zu bytes\n", arena.used(), arena.capacity());

    // ArenaScope — automatic rewind
    {
        ArenaScope scope(arena);
        char* temp = arena.alloc_array<char>(1024);
        std::printf("Inside scope: %zu bytes used\n", arena.used());
    }
    std::printf("After scope: %zu bytes used (rewound!)\n", arena.used());

    // Reset everything
    arena.reset();
    std::printf("After reset: %zu bytes used\n", arena.used());

    // Benchmark: arena vs new/delete
    constexpr int N = 1'000'000;

    auto start = std::chrono::high_resolution_clock::now();
    for (int i = 0; i < N; ++i) {
        int* p = arena.create<int>(i);
        if (arena.remaining() < sizeof(int)) arena.reset();
    }
    auto arena_time = std::chrono::high_resolution_clock::now() - start;

    start = std::chrono::high_resolution_clock::now();
    for (int i = 0; i < N; ++i) {
        int* p = new int(i);
        delete p;
    }
    auto new_time = std::chrono::high_resolution_clock::now() - start;

    std::printf("\nBenchmark (%d allocations):\n", N);
    std::printf("  Arena:      %lld µs\n",
        std::chrono::duration_cast<std::chrono::microseconds>(arena_time).count());
    std::printf("  new/delete: %lld µs\n",
        std::chrono::duration_cast<std::chrono::microseconds>(new_time).count());

    return 0;
}
```

<br>

#### Bonus Challenges

1. **Growable arena**: When the buffer is full, allocate a new larger block (see 9.9). Chain blocks via a linked list. `reset()` keeps the first block, frees the rest.

2. **`static_assert` for trivial types**: Add a `create()` overload that refuses non-trivially-destructible types, and a `create_tracked()` that accepts them but registers destructors.

3. **Thread-local scratch arena**: Create a `thread_local Arena` and an `ArenaScope`-based API. Benchmark versus `std::vector` temporaries.

4. **Arena-backed STL allocator**: Write an `ArenaAllocator<T>` that satisfies the STL allocator interface. Use it with `std::vector<int, ArenaAllocator<int>>`. (Preview of Day 11.)

<br><br>

---
---

# PART 7: DEEP DIVE — COMMON QUESTIONS

---
---

<br>

<h2 style="color: #2980B9;">📘 9.16 Q&A</h2>

<br>

#### Q1: "Why use `malloc` for the backing buffer instead of `new char[]`?"

`new char[]` gives you `char*` which works fine, but:
- `malloc` is simpler — no constructors, no `delete[]` needed, just `free()`
- For large allocations, you might want `mmap` instead (returns pages directly from the OS, can be returned with `munmap`)
- `malloc` is what real allocators use internally — it's the right abstraction level

<br>

#### Q2: "What about memory fragmentation?"

Arenas have **zero external fragmentation** during their lifetime — allocations are perfectly packed (except alignment padding, which is **internal** fragmentation).

Fragmentation only becomes a concern if you chain multiple arena blocks and they're different sizes scattered in the heap. But since arena blocks are large and few, this is rarely a problem.

<br>

#### Q3: "Can arenas be used with `std::vector`?"

Yes, with a custom allocator (Day 11). But note: `std::vector` calls `deallocate()` when resizing, which an arena ignores. The old buffer becomes wasted space inside the arena. For arena-backed containers, pre-reserve capacity to avoid unnecessary reallocations.

<br>

#### Q4: "Why not just use `alloca()` (stack allocation)?"

`alloca()` allocates on the **call stack**:
- Pro: Even faster than arena (just adjust stack pointer)
- Con: Limited by stack size (typically 1-8 MB)
- Con: Freed when function returns — can't pass to callers
- Con: No alignment control
- Con: Undefined behavior on failure (no error return — just stack overflow)

Arenas are heap-backed, so they can be much larger and have explicit lifetime control.

<br>

#### Q5: "Is an arena thread-safe?"

The basic implementation is NOT thread-safe. Options:
- **Thread-local arenas** (best — no synchronization needed)
- **Atomic bump**: use `std::atomic<size_t>` for `m_offset` with `fetch_add` — fast, but wastes padding due to races between align and allocate
- **Mutex**: works but defeats the purpose of being fast

Thread-local is the standard approach in game engines and compilers.

<br><br>

---

## Reflection Questions

1. What workloads in your past projects could have benefited from an arena?
2. Why is cache locality such a big performance win?
3. How does the arena relate to RAII? (Hint: `ArenaScope` is RAII for arena lifetime)
4. What are the tradeoffs of a growable arena vs a fixed-size one?
5. When would you NOT use an arena?

---

## Interview Questions

1. "What is an arena allocator? How does it work?"
2. "When would you use an arena allocator instead of `malloc`/`new`?"
3. "How do you handle alignment in a bump allocator?"
4. "What happens to objects with non-trivial destructors in an arena?"
5. "How does a frame allocator work in a game engine?"
6. "Compare arena vs pool allocator — when would you choose each?"
7. "How would you make an arena thread-safe? What's the best approach?"
8. "What is the `ArenaScope` pattern? How does it relate to RAII?"

---

**Next**: Day 10 — Pool Allocator →
