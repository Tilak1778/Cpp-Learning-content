# Day 11: STL Allocator Interface

[← Back to Study Plan](../lld-study-plan.md) | [← Day 10](day10-pool-allocator.md)

> **Time**: ~1.5-2 hours  
> **Goal**: Understand how STL containers obtain memory, learn the `Allocator` concept and `allocator_traits`, and wire your Day 10 pool allocator into `std::vector`.

---
---

# PART 1: WHY THE STL NEEDS AN ALLOCATOR INTERFACE

---
---

<br>

<h2 style="color: #2980B9;">📘 11.1 The Default Path — <code>std::allocator&lt;T&gt;</code></h2>

Every STL container has a hidden template parameter you rarely write:

```cpp
std::vector<int>                             // actually std::vector<int, std::allocator<int>>
std::map<std::string, int>                   // actually std::map<string, int, less<string>,
                                             //          allocator<pair<const string, int>>>
```

`std::allocator<T>` is the default. It calls `::operator new` / `::operator delete` under the hood — the general-purpose heap.

For most programs that's fine. But if you need:
- **Arena-backed** containers (bulk free at scope end)
- **Pool-backed** containers (fixed-size, cache-hot)
- **Shared-memory** containers (mmap regions)
- **Tracking** allocations (logging every malloc)

…you need to plug in a **custom allocator**.

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 11.2 The Allocator Concept (What the Container Expects)</h2>

A type `A` satisfies the **Allocator** concept if it provides (at minimum):

| Requirement | What the container calls |
|-------------|--------------------------|
| `A::value_type` | The element type `T` |
| `a.allocate(n)` | Return `T*` pointing to raw memory for `n` objects |
| `a.deallocate(p, n)` | Free the memory at `p` (which was from `allocate(n)`) |

That's it for a **minimal** allocator. The container does **not** call your allocator's constructor/destructor for elements — it uses **`allocator_traits`** to fill in everything else.

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 11.3 <code>std::allocator_traits</code> — The Adapter Layer</h2>

Containers never talk to your allocator directly for optional operations. They go through `std::allocator_traits<A>`, which provides **defaults** for anything your allocator doesn't define:

```
Container ──► allocator_traits<A> ──► Your Allocator A
                │
                │  Defaults if A doesn't provide:
                │  • construct()    → placement new
                │  • destroy()      → explicit destructor
                │  • max_size()     → numeric_limits
                │  • rebind         → template alias
                │  • propagate_on_* → false_type
                │  • is_always_equal → depends
```

This means your allocator only needs `value_type`, `allocate`, `deallocate`. Everything else has sensible defaults via `allocator_traits`.

<br>

#### What `allocator_traits` provides

| Trait / Method | Default if not in allocator | Purpose |
|----------------|----------------------------|---------|
| `pointer` | `T*` | Pointer type |
| `const_pointer` | `const T*` | Const pointer |
| `size_type` | `std::size_t` | Size representation |
| `difference_type` | `std::ptrdiff_t` | Pointer difference |
| `construct(a, p, args...)` | `::new((void*)p) T(args...)` | Placement new |
| `destroy(a, p)` | `p->~T()` | Explicit destructor call |
| `max_size(a)` | `numeric_limits<size_type>::max() / sizeof(T)` | Maximum allocation |
| `rebind_alloc<U>` | `A<U>` if `A` is a template; otherwise you must provide it | Allocate a different type |
| `propagate_on_container_copy_assignment` | `false_type` | Copy the allocator when container is copy-assigned? |
| `propagate_on_container_move_assignment` | `false_type` | Move the allocator when container is move-assigned? |
| `propagate_on_container_swap` | `false_type` | Swap allocators when containers are swapped? |
| `is_always_equal` | `true_type` if allocator is stateless | Can we always deallocate from one instance what was allocated by another? |

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 11.4 Rebinding — Why Containers Need It</h2>

You give `std::vector<int, MyAlloc<int>>` an allocator for **`int`**. But internally `std::list<int, MyAlloc<int>>` also needs to allocate **`Node<int>`** (a wrapper struct containing `int` + pointers). It must convert your `MyAlloc<int>` into a `MyAlloc<Node<int>>`.

This is **rebinding**:

```cpp
template <typename T>
struct MyAlloc {
    using value_type = T;

    // rebind: "give me the same allocator but for type U"
    template <typename U>
    struct rebind {
        using other = MyAlloc<U>;
    };

    // ...
};
```

Or, if `MyAlloc` is already a class template with one parameter, `allocator_traits` can auto-rebind without you writing `rebind` at all.

```
std::list<int, MyAlloc<int>>
  → internally needs MyAlloc<Node<int>>
  → uses allocator_traits<MyAlloc<int>>::rebind_alloc<Node<int>>
  → gets MyAlloc<Node<int>>
```

`std::vector` doesn't usually rebind (it allocates `T` directly), but `std::list`, `std::map`, `std::unordered_map` all do.

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 11.5 Stateful vs Stateless Allocators</h2>

**Stateless** (like `std::allocator`): every instance is interchangeable. Any `std::allocator<int>` can deallocate memory from any other `std::allocator<int>`. Mark with `is_always_equal = true_type`.

**Stateful** (like an arena-backed allocator): the allocator holds a pointer to a specific arena. You can only deallocate from the same arena that allocated. Mark with `is_always_equal = false_type`.

```cpp
// Stateless — no data members
template <typename T>
struct StatelessAlloc {
    using value_type = T;
    using is_always_equal = std::true_type;
    T* allocate(std::size_t n) { return static_cast<T*>(::operator new(n * sizeof(T))); }
    void deallocate(T* p, std::size_t) { ::operator delete(p); }
};

// Stateful — holds a pointer to a resource
template <typename T>
struct ArenaAlloc {
    using value_type = T;
    using is_always_equal = std::false_type;

    Arena* arena_;
    explicit ArenaAlloc(Arena* a) : arena_(a) {}

    T* allocate(std::size_t n) { return static_cast<T*>(arena_->alloc(n * sizeof(T), alignof(T))); }
    void deallocate(T*, std::size_t) { /* arena doesn't support individual free */ }
};
```

**Why it matters**: When you copy/move/swap containers with stateful allocators, the container must know whether to propagate (copy/move/swap) the allocator itself. That's what the `propagate_on_*` traits control.

<br><br>

---
---

# PART 2: BUILDING A MINIMAL CUSTOM ALLOCATOR

---
---

<br>

<h2 style="color: #2980B9;">📘 11.6 Minimal Allocator Template</h2>

Here is the absolute minimum to make `std::vector<T, MyAlloc<T>>` compile and work:

```cpp
#include <cstddef>
#include <limits>
#include <new>

template <typename T>
struct MinimalAlloc {
    using value_type = T;

    MinimalAlloc() = default;

    // Rebind constructor: allows MinimalAlloc<U> → MinimalAlloc<T>
    template <typename U>
    MinimalAlloc(const MinimalAlloc<U>&) noexcept {}

    T* allocate(std::size_t n) {
        if (n > std::numeric_limits<std::size_t>::max() / sizeof(T))
            throw std::bad_alloc();
        T* p = static_cast<T*>(::operator new(n * sizeof(T)));
        return p;
    }

    void deallocate(T* p, std::size_t /*n*/) noexcept {
        ::operator delete(p);
    }
};

// Two MinimalAlloc instances are always interchangeable (stateless)
template <typename T, typename U>
bool operator==(const MinimalAlloc<T>&, const MinimalAlloc<U>&) noexcept { return true; }

template <typename T, typename U>
bool operator!=(const MinimalAlloc<T>&, const MinimalAlloc<U>&) noexcept { return false; }
```

**Key points:**
- `value_type` — required
- `allocate(n)` — return `T*` for `n` objects
- `deallocate(p, n)` — free (the `n` is informational; you may ignore it)
- **Rebind constructor** `template<typename U> MinimalAlloc(const MinimalAlloc<U>&)` — needed so `allocator_traits::rebind_alloc` can convert between types
- **`operator==` / `!=`** — containers use these to check if two allocator instances can deallocate each other's memory

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 11.7 Using It with <code>std::vector</code></h2>

```cpp
#include <vector>
#include <iostream>

int main() {
    std::vector<int, MinimalAlloc<int>> v;
    v.push_back(10);
    v.push_back(20);
    v.push_back(30);

    for (int x : v) std::cout << x << " ";   // 10 20 30
    std::cout << "\n";
}
```

Under the hood, `vector` calls:
1. `allocator_traits<MinimalAlloc<int>>::allocate(alloc, n)` → your `allocate(n)`
2. `allocator_traits<MinimalAlloc<int>>::construct(alloc, ptr, value)` → placement new (default)
3. On resize: allocate new buffer, move elements, destroy old, deallocate old
4. `allocator_traits<MinimalAlloc<int>>::destroy(alloc, ptr)` → `ptr->~T()` (default)
5. `allocator_traits<MinimalAlloc<int>>::deallocate(alloc, ptr, n)` → your `deallocate(ptr, n)`

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 11.8 Adding Logging — Tracking Allocator</h2>

A practical first custom allocator: log every allocation and deallocation.

```cpp
template <typename T>
struct TrackingAlloc {
    using value_type = T;

    TrackingAlloc() = default;
    template <typename U>
    TrackingAlloc(const TrackingAlloc<U>&) noexcept {}

    T* allocate(std::size_t n) {
        std::size_t bytes = n * sizeof(T);
        T* p = static_cast<T*>(::operator new(bytes));
        std::printf("[alloc]   %zu objects (%zu bytes) at %p\n", n, bytes, (void*)p);
        return p;
    }

    void deallocate(T* p, std::size_t n) noexcept {
        std::printf("[dealloc] %zu objects (%zu bytes) at %p\n", n, n * sizeof(T), (void*)p);
        ::operator delete(p);
    }
};

template <typename T, typename U>
bool operator==(const TrackingAlloc<T>&, const TrackingAlloc<U>&) noexcept { return true; }
template <typename T, typename U>
bool operator!=(const TrackingAlloc<T>&, const TrackingAlloc<U>&) noexcept { return false; }
```

```cpp
int main() {
    std::vector<int, TrackingAlloc<int>> v;
    v.reserve(4);
    v.push_back(1);
    v.push_back(2);
    v.push_back(3);
    v.push_back(4);
    v.push_back(5);   // triggers reallocation
}
```

**Expected output** (sizes vary by implementation):

```
[alloc]   4 objects (16 bytes) at 0x...
[alloc]   8 objects (32 bytes) at 0x...       ← reallocation for 5th element
[dealloc] 4 objects (16 bytes) at 0x...       ← old buffer freed
[dealloc] 8 objects (32 bytes) at 0x...       ← final cleanup
```

This is extremely useful for understanding how `vector` grows internally.

<br><br>

---
---

# PART 3: WIRING YOUR POOL ALLOCATOR INTO STL

---
---

<br>

<h2 style="color: #2980B9;">📘 11.9 Wrapping the Pool for STL</h2>

Your Day 10 `PoolAllocator<T>` manages fixed-size blocks. To use it with `std::vector`, you need a thin **STL-conforming wrapper** that forwards to the pool:

```cpp
#include <cstddef>
#include <cassert>
#include <new>
#include <limits>

// Forward declare — your Day 10 pool
template <typename T> class PoolAllocator;

template <typename T>
struct PoolStlAlloc {
    using value_type = T;
    using is_always_equal = std::false_type;   // stateful: holds pool pointer

    PoolAllocator<T>* pool_;

    explicit PoolStlAlloc(PoolAllocator<T>* pool) noexcept : pool_(pool) {}

    template <typename U>
    PoolStlAlloc(const PoolStlAlloc<U>& other) noexcept : pool_(nullptr) {
        // Cross-type rebind: pool is T-specific, so this only works if U == T
        // For containers that don't rebind (vector), this is fine
        // For list/map (which rebind to Node<T>), you'd need a type-erased pool
        (void)other;
    }

    T* allocate(std::size_t n) {
        if (n != 1) throw std::bad_alloc();   // pool only does single-object alloc
        return static_cast<T*>(pool_->allocate());
    }

    void deallocate(T* p, std::size_t n) noexcept {
        assert(n == 1);
        pool_->deallocate(p);
    }
};

template <typename T, typename U>
bool operator==(const PoolStlAlloc<T>& a, const PoolStlAlloc<U>& b) noexcept {
    return a.pool_ == b.pool_;
}
template <typename T, typename U>
bool operator!=(const PoolStlAlloc<T>& a, const PoolStlAlloc<U>& b) noexcept {
    return !(a == b);
}
```

**Limitation**: `std::vector` allocates **N elements at once** (`allocate(n)` where `n > 1`). A pool allocator gives out **one block at a time**. So this wrapper only works for containers that allocate one element at a time (like `std::list`) or when you `reserve(1)` then only use one slot.

To use a pool with `std::vector`, you'd need a pool whose block size matches `vector`'s internal buffer, or use an arena-backed allocator instead (which handles any size).

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 11.10 Arena-Backed STL Allocator (Better Fit for vector)</h2>

An arena handles **any size**, making it a natural fit for `std::vector`:

```cpp
class Arena;   // your Day 9 arena

template <typename T>
struct ArenaStlAlloc {
    using value_type = T;
    using is_always_equal = std::false_type;

    Arena* arena_;

    explicit ArenaStlAlloc(Arena* a) noexcept : arena_(a) {}

    template <typename U>
    ArenaStlAlloc(const ArenaStlAlloc<U>& other) noexcept : arena_(other.arena_) {}

    T* allocate(std::size_t n) {
        void* p = arena_->alloc(n * sizeof(T), alignof(T));
        if (!p) throw std::bad_alloc();
        return static_cast<T*>(p);
    }

    void deallocate(T*, std::size_t) noexcept {
        // Arena doesn't support individual free — this is intentional.
        // Memory reclaimed on arena.reset().
    }
};

template <typename T, typename U>
bool operator==(const ArenaStlAlloc<T>& a, const ArenaStlAlloc<U>& b) noexcept {
    return a.arena_ == b.arena_;
}
template <typename T, typename U>
bool operator!=(const ArenaStlAlloc<T>& a, const ArenaStlAlloc<U>& b) noexcept {
    return !(a == b);
}
```

```cpp
int main() {
    Arena arena(1024 * 1024);   // 1 MB

    std::vector<int, ArenaStlAlloc<int>> v(ArenaStlAlloc<int>(&arena));
    v.reserve(1000);
    for (int i = 0; i < 1000; ++i) v.push_back(i);

    // Old buffers from vector resizing "leak" inside the arena — that's fine.
    // arena.reset() reclaims everything.
}
```

**Important subtlety**: When `vector` resizes, it calls `deallocate(old_buffer, old_n)`. Your arena's `deallocate` is a no-op, so the old buffer becomes **dead space** inside the arena. This is why you should `reserve()` upfront if possible — it avoids multiple grow-and-abandon cycles that waste arena space.

<br><br>

---
---

# PART 4: C++17 POLYMORPHIC MEMORY RESOURCES (<code>std::pmr</code>)

---
---

<br>

<h2 style="color: #2980B9;">📘 11.11 The Problem with Template Allocators</h2>

Classic STL allocators are **part of the type**:

```cpp
std::vector<int, std::allocator<int>>     // type A
std::vector<int, TrackingAlloc<int>>      // type B — COMPLETELY DIFFERENT TYPE

void process(std::vector<int>& v);        // only accepts type A
// cannot pass type B here!
```

This means:
- Functions must be templated on the allocator type, or you can't mix allocators
- Binary interfaces (libraries, ABIs) are locked to one allocator
- Changing the allocator changes every function signature

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 11.12 <code>std::pmr</code> — Type-Erased Allocators (C++17)</h2>

C++17 introduced `<memory_resource>` with a base class `std::pmr::memory_resource`:

```cpp
class memory_resource {
public:
    virtual ~memory_resource();
    void* allocate(std::size_t bytes, std::size_t alignment = alignof(std::max_align_t));
    void  deallocate(void* p, std::size_t bytes, std::size_t alignment = alignof(std::max_align_t));
    bool  is_equal(const memory_resource& other) const noexcept;

private:
    virtual void* do_allocate(std::size_t bytes, std::size_t alignment) = 0;
    virtual void  do_deallocate(void* p, std::size_t bytes, std::size_t alignment) = 0;
    virtual bool  do_is_equal(const memory_resource& other) const noexcept = 0;
};
```

Now all containers using `std::pmr::polymorphic_allocator<T>` share **one type**, regardless of the underlying resource:

```cpp
std::pmr::vector<int> v1(std::pmr::get_default_resource());    // heap
std::pmr::vector<int> v2(&my_arena_resource);                  // arena

// Both are std::pmr::vector<int> — same type!
void process(std::pmr::vector<int>& v);   // accepts either
```

<br>

#### Built-in memory resources

| Resource | Behavior |
|----------|----------|
| `std::pmr::new_delete_resource()` | Calls `::operator new` / `::operator delete` |
| `std::pmr::null_memory_resource()` | Always throws `bad_alloc` (useful for testing) |
| `std::pmr::monotonic_buffer_resource` | Arena-style bump allocator |
| `std::pmr::synchronized_pool_resource` | Thread-safe pool allocator |
| `std::pmr::unsynchronized_pool_resource` | Single-threaded pool allocator |

```cpp
// Arena-backed vector — single allocation, no individual free
char buffer[4096];
std::pmr::monotonic_buffer_resource arena(buffer, sizeof(buffer));
std::pmr::vector<int> v(&arena);
v.reserve(100);
for (int i = 0; i < 100; ++i) v.push_back(i);
```

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 11.13 Writing a Custom <code>memory_resource</code></h2>

Wrap your Day 9 arena as a `pmr` resource:

```cpp
#include <memory_resource>

class ArenaMemoryResource : public std::pmr::memory_resource {
    Arena& arena_;

public:
    explicit ArenaMemoryResource(Arena& a) : arena_(a) {}

private:
    void* do_allocate(std::size_t bytes, std::size_t alignment) override {
        void* p = arena_.alloc(bytes, alignment);
        if (!p) throw std::bad_alloc();
        return p;
    }

    void do_deallocate(void*, std::size_t, std::size_t) override {
        // Arena doesn't support individual free — no-op
    }

    bool do_is_equal(const memory_resource& other) const noexcept override {
        auto* o = dynamic_cast<const ArenaMemoryResource*>(&other);
        return o && (&o->arena_ == &arena_);
    }
};
```

```cpp
Arena arena(1024 * 1024);
ArenaMemoryResource resource(arena);

std::pmr::vector<int> v(&resource);
v.push_back(42);

std::pmr::string s(&resource);
s = "hello from arena";

// Both use the same arena — same type as any other pmr::vector<int> / pmr::string
```

<br><br>

---
---

# PART 5: BUILD EXERCISE

---
---

<br>

<h2 style="color: #2980B9;">📘 11.14 Exercise: Wire Your Pool/Arena into STL</h2>

Implement the following and test with `std::vector`:

<br>

### Part A: TrackingAlloc (warmup)

```cpp
template <typename T>
struct TrackingAlloc {
    using value_type = T;

    TrackingAlloc() = default;
    template <typename U>
    TrackingAlloc(const TrackingAlloc<U>&) noexcept {}

    T* allocate(std::size_t n);      // YOUR IMPL: ::operator new + log
    void deallocate(T* p, std::size_t n) noexcept;   // YOUR IMPL: ::operator delete + log
};
// + operator== / operator!=
```

Test: create `std::vector<int, TrackingAlloc<int>>`, push 10 elements, observe reallocation log.

<br>

### Part B: ArenaStlAlloc

```cpp
template <typename T>
struct ArenaStlAlloc {
    using value_type = T;
    using is_always_equal = std::false_type;

    Arena* arena_;

    explicit ArenaStlAlloc(Arena* a) noexcept;
    template <typename U>
    ArenaStlAlloc(const ArenaStlAlloc<U>& other) noexcept;

    T* allocate(std::size_t n);               // YOUR IMPL: arena_->alloc(...)
    void deallocate(T*, std::size_t) noexcept; // YOUR IMPL: no-op
};
// + operator== / operator!=
```

Test: create `std::vector<int, ArenaStlAlloc<int>>` on a 4 KB arena, push elements, print `arena.used()` after each push.

<br>

### Part C (bonus): `pmr` wrapper

```cpp
class ArenaMemoryResource : public std::pmr::memory_resource {
    // YOUR IMPL: wrap Arena, override do_allocate / do_deallocate / do_is_equal
};
```

Test: use `std::pmr::vector<int>` backed by your arena resource.

<br>

### Compile and run

```bash
g++ -std=c++17 -Wall -Wextra -fsanitize=address -o day11 day11_stl_alloc.cpp && ./day11
```

<br>

### Test driver sketch

```cpp
int main() {
    std::cout << "=== Part A: TrackingAlloc ===\n";
    {
        std::vector<int, TrackingAlloc<int>> v;
        for (int i = 0; i < 10; ++i) v.push_back(i);
        std::cout << "size=" << v.size() << " capacity=" << v.capacity() << "\n";
    }

    std::cout << "\n=== Part B: ArenaStlAlloc ===\n";
    {
        Arena arena(4096);
        std::vector<int, ArenaStlAlloc<int>> v(ArenaStlAlloc<int>(&arena));
        v.reserve(100);
        for (int i = 0; i < 100; ++i) v.push_back(i);
        std::cout << "vector size=" << v.size() << "\n";
        std::cout << "arena used=" << arena.used() << " / " << arena.capacity() << "\n";
    }

    std::cout << "\n=== Part C: pmr ArenaResource ===\n";
    {
        Arena arena(1024 * 1024);
        ArenaMemoryResource resource(arena);
        std::pmr::vector<int> v(&resource);
        for (int i = 0; i < 50; ++i) v.push_back(i);
        std::cout << "pmr vector size=" << v.size() << "\n";
    }

    std::cout << "\n=== Done ===\n";
}
```

<br>

#### Bonus Challenges

1. **Pool-backed `std::list`**: `std::list` allocates one node at a time — a perfect fit for your pool allocator. Write `PoolStlAlloc` and use it with `std::list<Widget, PoolStlAlloc<Widget>>`. Handle rebinding (the list allocates `Node<Widget>`, not `Widget`).

2. **Scoped arena allocator**: Combine `ArenaStlAlloc` with `ArenaScope` (Day 9). Build a temporary `vector` inside a scope — when the scope ends, the arena rewinds and all vector memory is reclaimed.

3. **Benchmark**: Compare `std::pmr::vector<int>` with `monotonic_buffer_resource` vs your arena resource vs default `new_delete_resource` for 1M pushes.

4. **Propagation traits**: Make your `ArenaStlAlloc` propagate on move assignment (`propagate_on_container_move_assignment = true_type`). Test `v1 = std::move(v2)` where `v1` and `v2` use different arenas — observe what happens.

<br><br>

---
---

# PART 6: DEEP DIVE — COMMON QUESTIONS

---
---

<br>

<h2 style="color: #2980B9;">📘 11.15 Q&A</h2>

<br>

#### Q1: "Why does `deallocate` take `n` if I already know the block size?"

The standard requires `deallocate(p, n)` where `n` matches the `allocate(n)` call. For general allocators (like `malloc`) the size is stored in metadata headers, so `n` is redundant. But for **arena/pool** allocators, knowing `n` at `deallocate` time can help: a sized-pool can route to the correct size-class bucket without looking up headers.

<br>

#### Q2: "Do I need `operator==`?"

Yes. Containers use `a1 == a2` to decide if memory from allocator `a1` can be freed by `a2`. For stateless allocators, always return `true`. For stateful (e.g. arena-backed), return `true` only if both point to the same resource.

When two allocators compare **unequal**, a container **cannot** simply steal the other's buffer on move — it must allocate new memory and copy elements. This is why `propagate_on_container_move_assignment` and `is_always_equal` matter for performance.

<br>

#### Q3: "Why is the rebind constructor needed?"

When `std::list<int, MyAlloc<int>>` needs to allocate `Node<int>`, it creates `MyAlloc<Node<int>>` from your `MyAlloc<int>`. The **rebind constructor** `template<typename U> MyAlloc(const MyAlloc<U>&)` makes this conversion possible. Without it, the container can't obtain an allocator for its internal node type.

<br>

#### Q4: "Classic allocator vs `pmr` — which should I use?"

| | Classic template allocator | `std::pmr` |
|---|---|---|
| Allocation type | Part of container type | Runtime polymorphism |
| Performance | Slightly faster (no virtual dispatch) | Virtual call per alloc/dealloc |
| Flexibility | Must template everything | Same type regardless of resource |
| Use case | Perf-critical inner loops | Library APIs, configurable resources |

For internal/perf-critical code, classic template allocators avoid virtual overhead. For APIs and configurable systems, `pmr` is more practical.

<br>

#### Q5: "What happens when `vector` resizes with an arena allocator?"

The old buffer's `deallocate` is a no-op (arena can't free individual chunks). The old buffer becomes **dead space** inside the arena. The new, larger buffer is allocated from the arena's current offset. This is why `reserve()` is important with arena-backed containers — it avoids repeated grow-and-abandon waste.

```
vector grow with arena:
  [old buf: 16B dead] [old buf: 32B dead] [current buf: 64B in use] [free...]
                                                                     ↑ offset
```

<br>

#### Q6: "Can I use different allocators for different containers sharing the same arena?"

Yes — that's the whole point. Multiple containers can use the same arena (or `memory_resource`), each with their own `ArenaStlAlloc<T>` instance pointing to the same `Arena*`. When the arena resets, all containers' memory is reclaimed at once.

<br><br>

---

## Reflection Questions

1. What is the minimum interface for a C++ allocator?
2. Why does `allocator_traits` exist instead of requiring allocators to define everything?
3. When would a stateful allocator cause problems with container move/swap?
4. What is the advantage of `std::pmr` over classic template allocators?
5. Why does `vector` waste arena space on resize?

---

## Interview Questions

1. "What is `std::allocator_traits` and why does it exist?"
2. "What is the minimal interface for a custom STL allocator?"
3. "How does rebinding work? Why is it needed?"
4. "What is `std::pmr::memory_resource`? How does it differ from template allocators?"
5. "Write a tracking allocator that logs every allocation for `std::vector`."
6. "How would you use an arena allocator with `std::vector`?"
7. "What is `propagate_on_container_move_assignment` and when does it matter?"
8. "What are the tradeoffs of stateful vs stateless allocators?"

---

**Next**: Day 12 — Review & Benchmark →
