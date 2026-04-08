# Day 8: How `new`/`delete` Work — operator new, Alignment & Placement New

[← Back to Study Plan](../lld-study-plan.md) | [← Day 6](day06-cow-pimpl.md)

> **Time**: ~1.5-2 hours
> **Goal**: Understand what really happens when you write `new T(args)` — it's NOT a single operation. Learn about `operator new` vs `malloc`, alignment requirements, placement new, and how to intercept allocations for debugging.

---
---

# PART 1: WHAT `new` ACTUALLY DOES

---
---

<br>

<h2 style="color: #2980B9;">📘 8.1 The Two Steps of `new`</h2>

When you write:

```cpp
Widget* w = new Widget(42, "hello");
```

The compiler translates this into **two distinct steps**:

```
Step 1: Allocate raw memory
  void* mem = operator new(sizeof(Widget));
  // Just raw bytes — no object exists yet!
  // Similar to malloc(sizeof(Widget))

Step 2: Construct the object in that memory
  Widget* w = new (mem) Widget(42, "hello");
  // This is "placement new" — constructs in-place
  // Calls Widget's constructor at address 'mem'
```

```
new Widget(42, "hello")  =  operator new(sizeof(Widget))  +  constructor call
─────────────────────       ──────────────────────────        ────────────────
  "new expression"           "allocation function"             "placement new"
  (you write this)           (gives raw bytes)                 (constructs object)
```

Similarly, `delete` is two steps in reverse:

```
delete w;

Step 1: Destroy the object
  w->~Widget();           // call destructor

Step 2: Free the raw memory
  operator delete(mem);   // return bytes to the heap
```

This separation is fundamental — it's why you can override allocation without changing construction, and why placement new exists.

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 8.2 <code>operator new</code> vs <code>malloc</code></h2>

Both allocate raw memory. But they are NOT the same:

| | `operator new` | `malloc` |
|---|---------------|----------|
| **Language** | C++ | C |
| **On failure** | Throws `std::bad_alloc` | Returns `nullptr` |
| **Overridable** | Yes — globally or per-class | No (but you can interpose with `LD_PRELOAD`) |
| **Calls constructor?** | No — just raw bytes | No |
| **Paired with** | `operator delete` | `free` |
| **Size parameter** | `size_t` (same) | `size_t` (same) |
| **New handler** | Calls `std::set_new_handler` callback before throwing | N/A |

Under the hood, the default `operator new` typically **calls `malloc`**:

```cpp
// Simplified default implementation (what the standard library provides):
void* operator new(std::size_t size) {
    if (size == 0) size = 1;  // C++ requires non-null for size 0
    
    while (true) {
        void* ptr = std::malloc(size);
        if (ptr) return ptr;
        
        // Allocation failed — try the new handler
        auto handler = std::get_new_handler();
        if (handler) {
            handler();  // might free memory, or throw, or abort
        } else {
            throw std::bad_alloc();
        }
    }
}

void operator delete(void* ptr) noexcept {
    std::free(ptr);
}
```

<br>

#### The `nothrow` variant

```cpp
Widget* w = new (std::nothrow) Widget();
// Returns nullptr on failure instead of throwing
// Uses: operator new(size, std::nothrow_t)
if (!w) { /* handle allocation failure */ }
```

<br>

#### Never mix them!

```cpp
int* p = (int*)malloc(sizeof(int));
delete p;       // UNDEFINED BEHAVIOR — malloc paired with delete

int* q = new int;
free(q);        // UNDEFINED BEHAVIOR — new paired with free
```

The internal bookkeeping can differ (e.g., `new` might store the size before the pointer for array `delete[]`). Mixing them corrupts the heap.

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 8.3 The `new` / `delete` Family</h2>

There are multiple forms of allocation functions:

```cpp
// ── Single object ──
void* operator new(std::size_t size);                         // throwing
void* operator new(std::size_t size, std::nothrow_t) noexcept; // non-throwing
void  operator delete(void* ptr) noexcept;
void  operator delete(void* ptr, std::size_t size) noexcept;  // sized delete (C++14)

// ── Arrays ──
void* operator new[](std::size_t size);
void* operator new[](std::size_t size, std::nothrow_t) noexcept;
void  operator delete[](void* ptr) noexcept;
void  operator delete[](void* ptr, std::size_t size) noexcept;

// ── Placement (non-allocating) ──
void* operator new(std::size_t size, void* ptr) noexcept;     // returns ptr as-is
void  operator delete(void* ptr, void* place) noexcept;       // no-op
```

**Critical rule**: `new[]` must be paired with `delete[]`, and `new` with `delete`:

```cpp
int* arr = new int[10];
delete arr;      // UNDEFINED BEHAVIOR — should be delete[]
delete[] arr;    // correct

Widget* w = new Widget();
delete[] w;      // UNDEFINED BEHAVIOR — should be delete
delete w;        // correct
```

Why? `new[]` stores the array count (typically just before the returned pointer) so `delete[]` knows how many destructors to call. `delete` doesn't read this count.

<br><br>

---
---

# PART 2: ALIGNMENT

---
---

<br>

<h2 style="color: #2980B9;">📘 8.4 What Is Alignment?</h2>

Every type has an **alignment requirement** — the address where it can be placed must be a multiple of its alignment:

```cpp
alignof(char)   == 1   // can be at any address
alignof(short)  == 2   // address must be multiple of 2
alignof(int)    == 4   // address must be multiple of 4
alignof(double) == 8   // address must be multiple of 8
alignof(void*)  == 8   // on 64-bit systems
```

Why? The CPU reads memory in aligned chunks. Misaligned access is either:
- **Slower** (x86: hardware fixes it with extra cycles)
- **A crash** (ARM, SPARC: `SIGBUS`)

```
Memory addresses:
 0   1   2   3   4   5   6   7   8   9  10  11  12
┌───┬───┬───┬───┬───┬───┬───┬───┬───┬───┬───┬───┬───┐
│   │   │   │   │   │   │   │   │   │   │   │   │   │

int (4 bytes, alignof=4):
  Valid:   address 0, 4, 8, 12, ...
  Invalid: address 1, 2, 3, 5, 6, 7, ...

double (8 bytes, alignof=8):
  Valid:   address 0, 8, 16, 24, ...
  Invalid: address 1-7, 9-15, ...
```

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 8.5 Struct Padding and <code>sizeof</code></h2>

The compiler inserts **padding bytes** to satisfy alignment:

```cpp
struct Bad {
    char  a;   // 1 byte  + 3 bytes padding (next field needs 4-byte alignment)
    int   b;   // 4 bytes
    char  c;   // 1 byte  + 7 bytes padding (struct alignment = alignof(double) = 8)
    double d;  // 8 bytes
};
// sizeof(Bad) = 24  (only 14 bytes of actual data!)
// alignof(Bad) = 8  (alignment of the largest member)

struct Good {
    double d;  // 8 bytes (already aligned)
    int    b;  // 4 bytes
    char   a;  // 1 byte
    char   c;  // 1 byte + 2 bytes padding (struct size must be multiple of alignof=8)
};
// sizeof(Good) = 16  (saved 8 bytes just by reordering!)
// alignof(Good) = 8
```

```
Memory layout of Bad:
┌───┬───────┬────────┬───┬───────────────┬────────────────┐
│ a │ pad   │   b    │ c │     pad       │       d        │
│ 1 │  3    │   4    │ 1 │      7        │       8        │
└───┴───────┴────────┴───┴───────────────┴────────────────┘
 0          4         8                  16              24

Memory layout of Good:
┌────────────────┬────────┬───┬───┬─────┐
│       d        │   b    │ a │ c │ pad │
│       8        │   4    │ 1 │ 1 │  2  │
└────────────────┴────────┴───┴───┴─────┘
 0               8       12 13 14  16
```

**Rule of thumb**: order struct members from largest to smallest alignment to minimize padding.

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 8.6 <code>alignas</code> — Requesting Stricter Alignment</h2>

You can request alignment stricter than the type's natural requirement:

```cpp
// Cache line is typically 64 bytes on modern CPUs
struct alignas(64) CacheAlignedCounter {
    std::atomic<int> count{0};
    // Padded to 64 bytes automatically
};
// sizeof(CacheAlignedCounter) == 64
// alignof(CacheAlignedCounter) == 64

CacheAlignedCounter counters[4];
// Each counter starts at a 64-byte boundary
// → No false sharing between threads!
```

**False sharing**: when two threads modify different variables that happen to be on the **same cache line**, each write invalidates the other CPU's cache — massive slowdown. Aligning to cache-line boundaries prevents this.

```
Without alignas(64):
┌─── Cache Line (64 bytes) ───────────────────────────┐
│ counter[0] │ counter[1] │ ... (other data)           │
└─────────────────────────────────────────────────────┘
Thread A writes counter[0] → invalidates Thread B's cache line
Thread B writes counter[1] → invalidates Thread A's cache line
→ "False sharing" — ping-pong between cores

With alignas(64):
┌─── Cache Line 0 ──────────────────────────────────┐
│ counter[0]                                         │
└────────────────────────────────────────────────────┘
┌─── Cache Line 1 ──────────────────────────────────┐
│ counter[1]                                         │
└────────────────────────────────────────────────────┘
→ No false sharing — each counter on its own cache line
```

<br>

#### `alignas` rules

```cpp
alignas(32) int x;             // x aligned to 32 bytes
alignas(double) int y;         // y aligned like a double (8 bytes)
alignas(1) double z;           // ERROR — can't weaken alignment below natural
alignas(0) int w;              // ignored (no effect)
```

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 8.7 Over-Aligned Types and <code>operator new</code></h2>

The default `operator new` guarantees alignment of `__STDCPP_DEFAULT_NEW_ALIGNMENT__` (typically 16 bytes on 64-bit). If your type needs **more** than that:

```cpp
struct alignas(64) BigAligned {
    char data[64];
};

// Pre-C++17: operator new returns 16-byte aligned memory
// BigAligned needs 64-byte alignment → UNDEFINED BEHAVIOR!

// C++17 added aligned new:
BigAligned* p = new BigAligned;  // compiler calls:
// operator new(sizeof(BigAligned), std::align_val_t(64))
// → allocator knows to return 64-byte aligned memory
```

If you're overriding `operator new` for over-aligned types, you must also override the aligned variant:

```cpp
struct alignas(64) MyType {
    int data;
    
    static void* operator new(std::size_t size, std::align_val_t align) {
        void* ptr = std::aligned_alloc(static_cast<std::size_t>(align), size);
        if (!ptr) throw std::bad_alloc();
        return ptr;
    }
    
    static void operator delete(void* ptr, std::size_t size, std::align_val_t align) {
        std::free(ptr);
    }
};
```

<br><br>

---
---

# PART 3: PLACEMENT NEW

---
---

<br>

<h2 style="color: #2980B9;">📘 8.8 What Is Placement New?</h2>

Placement new **constructs an object at a specific memory address** without allocating anything. You provide the memory, placement new calls the constructor:

```cpp
#include <new>  // needed for placement new

// Step 1: Get some memory (from anywhere — stack, arena, mmap, etc.)
alignas(Widget) char buffer[sizeof(Widget)];

// Step 2: Construct an object IN that memory
Widget* w = new (buffer) Widget(42, "hello");
//            ^^^^^^^^ — this is the "placement" address

// Step 3: Use the object normally
w->doSomething();

// Step 4: Destroy the object MANUALLY (no delete!)
w->~Widget();  // explicit destructor call — the ONLY time you call destructors directly

// DO NOT call delete w — you didn't allocate with operator new!
// The buffer is stack memory, it goes away when the scope ends.
```

**Placement new syntax**: `new (address) Type(args...)`

The `operator new` overload for placement new is trivial — it just returns the address you gave it:

```cpp
void* operator new(std::size_t, void* ptr) noexcept {
    return ptr;  // no allocation — just returns the address
}
```

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 8.9 Why Placement New Matters</h2>

Placement new is the backbone of many C++ idioms:

<br>

#### 1. How `std::vector` works internally

```cpp
// Simplified vector push_back:
template<typename T>
void vector<T>::push_back(const T& value) {
    if (m_size == m_capacity) {
        // Allocate new raw memory (no constructors called)
        T* newData = static_cast<T*>(operator new(m_capacity * 2 * sizeof(T)));
        
        // Move-construct existing elements into new memory
        for (size_t i = 0; i < m_size; ++i) {
            new (&newData[i]) T(std::move(m_data[i]));  // placement new
            m_data[i].~T();                              // destroy old
        }
        
        operator delete(m_data);  // free old raw memory
        m_data = newData;
        m_capacity *= 2;
    }
    
    // Construct the new element in-place
    new (&m_data[m_size]) T(value);  // placement new
    ++m_size;
}
```

The key insight: `vector` separates **capacity** (raw memory) from **size** (constructed objects). It uses placement new to construct objects one by one in pre-allocated memory.

<br>

#### 2. Custom allocators (Arena, Pool — Days 9-10)

```cpp
class Arena {
    char* m_buffer;
    size_t m_offset = 0;
public:
    template<typename T, typename... Args>
    T* create(Args&&... args) {
        // Align the offset
        m_offset = align_up(m_offset, alignof(T));
        
        // Get pointer into the arena
        void* ptr = m_buffer + m_offset;
        m_offset += sizeof(T);
        
        // Construct in-place with placement new
        return new (ptr) T(std::forward<Args>(args)...);
    }
};
```

<br>

#### 3. `std::optional` / `std::variant` (stack storage of possibly-uninitialized objects)

```cpp
// Simplified optional:
template<typename T>
class Optional {
    alignas(T) char m_storage[sizeof(T)];  // raw bytes, aligned for T
    bool m_has_value = false;

public:
    template<typename... Args>
    void emplace(Args&&... args) {
        if (m_has_value) {
            reinterpret_cast<T*>(m_storage)->~T();  // destroy old
        }
        new (m_storage) T(std::forward<Args>(args)...);  // construct new
        m_has_value = true;
    }

    ~Optional() {
        if (m_has_value) {
            reinterpret_cast<T*>(m_storage)->~T();
        }
    }
};
```

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 8.10 Alignment Pitfalls with Placement New</h2>

The memory you provide to placement new **must be properly aligned**:

```cpp
// WRONG — char array has alignment 1, Widget may need alignment 8
char buffer[sizeof(Widget)];
Widget* w = new (buffer) Widget();  // potential SIGBUS on ARM!

// CORRECT — use alignas
alignas(Widget) char buffer[sizeof(Widget)];
Widget* w = new (buffer) Widget();  // guaranteed correct alignment

// ALSO CORRECT — use std::aligned_storage (pre-C++23) or unions
std::aligned_storage_t<sizeof(Widget), alignof(Widget)> buffer;
Widget* w = new (&buffer) Widget();
```

<br><br>

---
---

# PART 4: OVERRIDING `operator new` / `operator delete`

---
---

<br>

<h2 style="color: #2980B9;">📘 8.11 Global Override — Intercepting All Allocations</h2>

You can replace the global `operator new` and `operator delete` for the entire program:

```cpp
#include <cstdlib>
#include <cstdio>
#include <new>

struct AllocStats {
    size_t total_allocations = 0;
    size_t total_deallocations = 0;
    size_t total_bytes_allocated = 0;
    size_t current_bytes = 0;
    size_t peak_bytes = 0;
};

static AllocStats g_stats;

void* operator new(std::size_t size) {
    g_stats.total_allocations++;
    g_stats.total_bytes_allocated += size;
    g_stats.current_bytes += size;
    if (g_stats.current_bytes > g_stats.peak_bytes) {
        g_stats.peak_bytes = g_stats.current_bytes;
    }

    // Store the size just before the returned pointer so we know it in delete
    void* raw = std::malloc(size + sizeof(std::size_t));
    if (!raw) throw std::bad_alloc();

    *static_cast<std::size_t*>(raw) = size;
    void* user_ptr = static_cast<char*>(raw) + sizeof(std::size_t);

    std::printf("[new]    %zu bytes at %p\n", size, user_ptr);
    return user_ptr;
}

void operator delete(void* ptr) noexcept {
    if (!ptr) return;

    void* raw = static_cast<char*>(ptr) - sizeof(std::size_t);
    std::size_t size = *static_cast<std::size_t*>(raw);

    g_stats.total_deallocations++;
    g_stats.current_bytes -= size;

    std::printf("[delete] %zu bytes at %p\n", size, ptr);
    std::free(raw);
}

void operator delete(void* ptr, std::size_t size) noexcept {
    // Sized delete (C++14) — size provided by the compiler
    operator delete(ptr);  // delegate to unsized version
}

void print_alloc_stats() {
    std::printf("\n=== Allocation Stats ===\n");
    std::printf("Total allocations:   %zu\n", g_stats.total_allocations);
    std::printf("Total deallocations: %zu\n", g_stats.total_deallocations);
    std::printf("Total bytes alloc'd: %zu\n", g_stats.total_bytes_allocated);
    std::printf("Current bytes:       %zu\n", g_stats.current_bytes);
    std::printf("Peak bytes:          %zu\n", g_stats.peak_bytes);
    if (g_stats.total_allocations != g_stats.total_deallocations) {
        std::printf("WARNING: potential memory leak!\n");
    }
}
```

**Why `printf` instead of `std::cout`?** — `std::cout` may call `operator new` internally (for locale, buffer management), causing infinite recursion. `printf` uses `malloc` directly, which we're not overriding.

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 8.12 Per-Class Override</h2>

You can override `operator new` / `delete` for a **single class** — useful for pool allocation:

```cpp
class Particle {
    float x, y, z;
    float vx, vy, vz;

    static void* s_pool;
    static std::vector<void*> s_free_list;

public:
    static void* operator new(std::size_t size) {
        if (s_free_list.empty()) {
            // Fallback to global new
            return ::operator new(size);
        }
        void* ptr = s_free_list.back();
        s_free_list.pop_back();
        return ptr;
    }

    static void operator delete(void* ptr) noexcept {
        s_free_list.push_back(ptr);
    }
};

// Per-class operator new is called ONLY for:
Particle* p = new Particle();     // uses Particle::operator new
Particle* q = new Particle[10];   // uses GLOBAL operator new[] (unless you also override new[])
```

The `static` is implicit — class-scope `operator new`/`delete` are always static (they run before/after the object's lifetime, so there's no `this`).

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 8.13 <code>std::set_new_handler</code> — Custom Failure Behavior</h2>

Before throwing `std::bad_alloc`, `operator new` calls the **new handler** in a loop:

```cpp
void my_new_handler() {
    std::printf("Memory allocation failed — trying to free cache...\n");
    
    // Option 1: Free some memory and return → operator new retries
    global_cache.clear();
    
    // Option 2: Throw an exception (not necessarily bad_alloc)
    // throw MyMemoryException();
    
    // Option 3: Abort
    // std::abort();
    
    // Option 4: Set handler to nullptr → next failure throws bad_alloc
    // std::set_new_handler(nullptr);
}

int main() {
    std::set_new_handler(my_new_handler);
    // Now if any allocation fails, my_new_handler runs first
}
```

This is rarely used in practice, but it's a common interview question.

<br><br>

---
---

# PART 5: BUILD EXERCISE

---
---

<br>

<h2 style="color: #2980B9;">📘 8.14 Exercise: Allocation Tracker</h2>

Build a global allocation tracker that overrides `operator new` / `operator delete` and provides:

1. **Logging**: Print every allocation and deallocation with size and address
2. **Statistics**: Track total allocations, deallocations, bytes allocated, peak usage
3. **Leak detection**: At program exit, report any unfreed allocations with their addresses and sizes

<br>

#### Skeleton

```cpp
// alloc_tracker.h
#pragma once
#include <cstddef>

struct AllocRecord {
    void* address;
    std::size_t size;
    // Optionally: file, line (use macros)
};

class AllocTracker {
public:
    static AllocTracker& instance();

    void record_alloc(void* ptr, std::size_t size);
    void record_dealloc(void* ptr);
    void print_stats() const;
    void print_leaks() const;

private:
    // Use a data structure that does NOT call operator new internally!
    // Options: fixed-size array, or mmap-based storage
    // If you use std::map/vector, you'll get infinite recursion.
};

// alloc_tracker.cpp — implement the tracker

// alloc_overrides.cpp — override operator new/delete, delegate to tracker

// main.cpp — test with various allocations:
//   int* p = new int(42);
//   std::string s = "hello world, this is a long string";
//   std::vector<int> v(1000);
//   // intentionally leak one:
//   int* leak = new int[100];
//   // At exit: print_stats() and print_leaks()
```

<br>

#### Requirements

- Override **all four** forms: `new`, `delete`, `new[]`, `delete[]`
- Use `printf`/`fprintf` (not `cout`) in the overrides to avoid recursion
- Store allocation records in a **fixed-size array** (e.g., 10000 entries) to avoid calling `new` inside `new`
- At exit, print a summary and list any leaked allocations
- Compile with: `g++ -std=c++17 -Wall -Wextra -o tracker alloc_tracker.cpp alloc_overrides.cpp main.cpp`

<br>

#### Bonus Challenges

1. **Source location tracking**: Use `__FILE__` and `__LINE__` macros with a `#define new` trick (dangerous but educational) or C++20 `std::source_location`
2. **Thread safety**: Protect the tracker with a spinlock (don't use `std::mutex` — its constructor may call `operator new`)
3. **Double-free detection**: Check if a pointer being freed was never allocated (or already freed)
4. **Stack traces**: Capture backtrace on allocation using `backtrace()` (POSIX) to find where leaks were created

<br><br>

---
---

# PART 6: DEEP DIVE — COMMON QUESTIONS

---
---

<br>

<h2 style="color: #2980B9;">📘 8.15 Q&A: Tricky Corners</h2>

<br>

#### Q1: "Can I call `delete` on a `void*`?"

```cpp
void* ptr = new Widget();
delete ptr;  // UNDEFINED BEHAVIOR!
```

`delete` needs to call the destructor, but it doesn't know the type of the object behind `void*`. The destructor is never called — only the memory is freed. This is a silent bug.

<br>

#### Q2: "What happens if I call `delete` on `nullptr`?"

```cpp
delete nullptr;  // perfectly safe — defined as a no-op by the standard
```

<br>

#### Q3: "What's the difference between `operator new` and `new` expression?"

```
new expression:    Widget* w = new Widget(42);
                   → calls operator new(sizeof(Widget))
                   → calls Widget::Widget(42)
                   → returns Widget*

operator new:      void* raw = operator new(sizeof(Widget));
                   → just allocates raw bytes
                   → no constructor called
                   → returns void*
```

You almost never call `operator new` directly — only in custom allocators or when implementing placement new patterns.

<br>

#### Q4: "What is `std::aligned_alloc`?"

C11/C++17 function that allocates memory with a specific alignment:

```cpp
// Allocate 1024 bytes aligned to a 64-byte boundary
void* ptr = std::aligned_alloc(64, 1024);
// NOTE: size must be a multiple of alignment!
std::free(ptr);
```

On macOS, `posix_memalign` is more portable:

```cpp
void* ptr;
posix_memalign(&ptr, 64, 1024);  // any size works
std::free(ptr);
```

<br>

#### Q5: "What does `new int()` vs `new int` do?"

```cpp
int* a = new int;    // uninitialized — contains garbage
int* b = new int();  // value-initialized — guaranteed to be 0
int* c = new int{};  // value-initialized — guaranteed to be 0 (C++11)

Widget* w1 = new Widget;    // default-initialized (calls default ctor)
Widget* w2 = new Widget();  // value-initialized (also calls default ctor for class types)
// For class types with user-defined constructors, no difference.
// The difference only matters for POD/scalar types.
```

<br>

#### Q6: "How does `delete[]` know the array size?"

The implementation stores the count **before the returned pointer**:

```
operator new[](5 * sizeof(Widget)):

  What the allocator returns:
  ┌──────────┬─────────┬─────────┬─────────┬─────────┬─────────┐
  │ count: 5 │ Widget  │ Widget  │ Widget  │ Widget  │ Widget  │
  └──────────┴─────────┴─────────┴─────────┴─────────┴─────────┘
  ↑            ↑
  raw pointer  pointer returned to you

  delete[]:
  1. Read count (5) from just before the pointer
  2. Call ~Widget() on each element (in reverse order)
  3. Free the raw memory (including the count prefix)
```

This is why `delete` (without `[]`) on an array is UB — it doesn't read the count, calls only one destructor, and may pass the wrong address to `free`.

<br><br>

---

## Reflection Questions

1. When would you override `operator new` globally? When per-class?
2. Why is placement new essential for writing `std::vector`?
3. How does `alignas(64)` prevent false sharing?
4. Why must you manually call the destructor after placement new?
5. What goes wrong if you `delete` what was allocated with `malloc`?

---

## Interview Questions

1. "What happens when you write `new Widget(42)`? Walk through every step."
2. "What is placement new? When would you use it?"
3. "How would you implement a memory leak detector?"
4. "What is alignment? Why does it matter? What is `alignas`?"
5. "Why does `delete[]` exist as separate from `delete`?"
6. "How does `std::vector` separate memory allocation from object construction?"
7. "What is the new handler? When does it run?"
8. "Can you override `operator new` for a single class? How?"

---

**Next**: Day 9 — Arena (Bump) Allocator →
