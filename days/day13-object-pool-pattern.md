# Day 13: Object Pool Pattern

[← Back to Study Plan](../lld-study-plan.md) | [← Day 12](day12-review-and-benchmark.md)

> **Time**: ~2-3 hours (weekend day)  
> **Goal**: Understand the Object Pool design pattern — **reuse** expensive-to-create objects instead of constructing/destroying them repeatedly. Build a generic `ObjectPool<T>` with `acquire()` / `release()` semantics, RAII-safe handles, and thread-safety awareness.

---

---

# PART 1: THE PROBLEM — EXPENSIVE CONSTRUCTION

---

---

  


## 📘 13.1 When Allocation Isn't the Bottleneck

Days 9-12 focused on making **memory allocation** faster. But sometimes the bottleneck isn't `malloc` — it's **constructing the object itself**:


| Object                     | Why construction is expensive                                        |
| -------------------------- | -------------------------------------------------------------------- |
| Database connection        | TCP handshake, authentication, SSL negotiation (~50-200 ms)          |
| Thread                     | OS kernel call to create stack, register with scheduler (~10-100 µs) |
| GPU buffer                 | Driver call, memory mapping, synchronization (~1-10 ms)              |
| SSL/TLS context            | Key generation, certificate loading (~5-50 ms)                       |
| Large pre-allocated buffer | Zeroing or filling MBs of memory                                     |
| Compiled regex             | Parse + compile pattern to state machine                             |


For these objects, you don't want to pay construction cost **every time** you need one. Instead: **create them once, reuse them many times**.

```
Without pool:
  request 1: create connection (200ms) → use (5ms) → destroy (10ms)
  request 2: create connection (200ms) → use (5ms) → destroy (10ms)
  request 3: create connection (200ms) → use (5ms) → destroy (10ms)
  Total: 645ms for 3 requests

With pool:
  startup:   create 3 connections (600ms one-time cost)
  request 1: acquire (0.001ms) → use (5ms) → release (0.001ms)
  request 2: acquire (0.001ms) → use (5ms) → release (0.001ms)
  request 3: acquire (0.001ms) → use (5ms) → release (0.001ms)
  Total: 615ms, but requests 2 & 3 save ~200ms each
```

  
  


---

  


## 📘 13.2 Object Pool vs Pool Allocator — Different Things

This is a common source of confusion:


|                       | Day 10: Pool **Allocator**               | Day 13: Object **Pool**                                                                   |
| --------------------- | ---------------------------------------- | ----------------------------------------------------------------------------------------- |
| **Level**             | Memory management (raw bytes)            | Application-level design pattern                                                          |
| **What it manages**   | Fixed-size **memory blocks**             | Fully **constructed objects**                                                             |
| `**acquire`** returns | Raw memory → you placement-new           | A **ready-to-use** object                                                                 |
| `**release`** does    | Returns memory block to free list        | Returns object to pool — object stays alive, may be **reset**                             |
| **Object lifetime**   | Created on acquire, destroyed on release | Created at pool startup (or on first need), **reused** across many acquire/release cycles |
| **Why it's fast**     | No malloc overhead                       | No **construction** overhead                                                              |


A pool allocator makes `new`/`delete` faster. An object pool makes **expensive constructors** cheaper by avoiding them altogether.

  
  


---

---

# PART 2: THE PATTERN

---

---

  


## 📘 13.3 Core API: `acquire()` and `release()`

```cpp
template <typename T>
class ObjectPool {
public:
    // Get an object from the pool (or create one if pool is empty)
    T* acquire();

    // Return an object to the pool for reuse
    void release(T* obj);
};
```

The lifecycle:

```
                    ┌────────────────────────────┐
                    │       Object Pool          │
                    │  ┌────┐ ┌────┐ ┌────┐     │
                    │  │ T1 │ │ T2 │ │ T3 │     │  ← pre-constructed, idle
                    │  └────┘ └────┘ └────┘     │
                    └────────────────────────────┘
                         │
            acquire()    │    release()
                 ↓       │       ↑
              ┌──────────┴───────┴──┐
              │    Client Code      │
              │    uses T object    │
              └─────────────────────┘
```

**Contract:**

- `acquire()` gives you a valid, ready-to-use `T`*
- You **must** call `release()` when done (or use an RAII handle)
- After `release()`, you **must not** use the pointer — someone else may `acquire()` it next
- The pool may **reset** the object on `release()` (or `acquire()`) to prepare it for the next user

  
  


---

  


## 📘 13.4 Growth Strategies

What happens when `acquire()` is called but the pool is empty?


| Strategy           | Behavior                    | Trade-off                                                     |
| ------------------ | --------------------------- | ------------------------------------------------------------- |
| **Fixed size**     | Throw/return null           | Predictable memory, but callers must handle failure           |
| **On-demand grow** | Create a new `T`            | Flexible, but occasional latency spike on creation            |
| **Pre-warm**       | Create N objects at startup | Avoids runtime spikes, but may waste memory if N is too large |
| **Bounded grow**   | Grow up to max, then fail   | Balances flexibility and resource limits                      |


Most production pools use **bounded grow with pre-warming**: create a reasonable initial batch, grow on demand up to a cap, and block or fail beyond that.

  
  


---

  


## 📘 13.5 Reset on Return

When an object is returned to the pool, it may carry state from the previous user. You need to **reset** it before the next user gets it:

```cpp
// Option A: Reset on release (caller's state is cleaned up immediately)
void release(T* obj) {
    obj->reset();           // clear previous state
    available_.push(obj);
}

// Option B: Reset on acquire (lazy — only clean if someone actually reuses it)
T* acquire() {
    T* obj = available_.top();
    available_.pop();
    obj->reset();           // clean before handing out
    return obj;
}

// Option C: Require T to implement a Reusable interface
class Reusable {
public:
    virtual void reset() = 0;
    virtual ~Reusable() = default;
};
```

**Option A** (reset on release) is usually preferred — it prevents stale state from leaking between users and keeps the pool in a "clean" state.

  
  


---

  


## 📘 13.6 RAII Handle — Automatic Release

Manually calling `release()` is error-prone (just like manual `delete`). Use an RAII wrapper:

```cpp
template <typename T>
class PoolHandle {
    ObjectPool<T>* pool_;
    T* obj_;

public:
    PoolHandle(ObjectPool<T>* pool, T* obj) : pool_(pool), obj_(obj) {}
    ~PoolHandle() { if (obj_) pool_->release(obj_); }

    PoolHandle(const PoolHandle&) = delete;
    PoolHandle& operator=(const PoolHandle&) = delete;

    PoolHandle(PoolHandle&& other) noexcept : pool_(other.pool_), obj_(other.obj_) {
        other.obj_ = nullptr;
    }
    PoolHandle& operator=(PoolHandle&& other) noexcept {
        if (this != &other) {
            if (obj_) pool_->release(obj_);
            pool_ = other.pool_;
            obj_ = other.obj_;
            other.obj_ = nullptr;
        }
        return *this;
    }

    T& operator*() const { return *obj_; }
    T* operator->() const { return obj_; }
    T* get() const { return obj_; }
};
```

Now usage becomes leak-proof:

```cpp
{
    auto handle = pool.acquire_handle();   // returns PoolHandle<T>
    handle->doWork();
}   // automatically returned to pool — no manual release()
```

This mirrors the same RAII principle as `unique_ptr` (Day 2) and `ScopeGuard` (Day 5).

  
  


---

---

# PART 3: IMPLEMENTATION

---

---

  


## 📘 13.7 Design Decisions

Before writing code, decide:


| Decision          | Options                                                                                 |
| ----------------- | --------------------------------------------------------------------------------------- |
| **Storage**       | `std::vector<std::unique_ptr<T>>` for all objects + `std::stack<T*>` for available ones |
| **Growth**        | On-demand with optional max cap                                                         |
| **Reset**         | Require `T` to have a `reset()` method, or accept a reset function                      |
| **Thread safety** | Single-threaded first, then add mutex (or lock-free)                                    |
| **Handle**        | Return raw `T`* + manual release, or `PoolHandle<T>` with RAII                          |


  
  


---

  


## 📘 13.8 Complete Implementation

```cpp
#include <iostream>
#include <vector>
#include <stack>
#include <memory>
#include <functional>
#include <stdexcept>
#include <utility>

template <typename T>
class ObjectPool;

// ── RAII Handle ─────────────────────────────────────────

template <typename T>
class PoolHandle {
    ObjectPool<T>* pool_;
    T* obj_;

public:
    PoolHandle() : pool_(nullptr), obj_(nullptr) {}
    PoolHandle(ObjectPool<T>* pool, T* obj) : pool_(pool), obj_(obj) {}

    ~PoolHandle() {
        if (obj_ && pool_) pool_->release(obj_);
    }

    PoolHandle(const PoolHandle&) = delete;
    PoolHandle& operator=(const PoolHandle&) = delete;

    PoolHandle(PoolHandle&& other) noexcept
        : pool_(other.pool_), obj_(other.obj_) {
        other.obj_ = nullptr;
    }

    PoolHandle& operator=(PoolHandle&& other) noexcept {
        if (this != &other) {
            if (obj_ && pool_) pool_->release(obj_);
            pool_ = other.pool_;
            obj_ = other.obj_;
            other.obj_ = nullptr;
        }
        return *this;
    }

    T& operator*() const { return *obj_; }
    T* operator->() const { return obj_; }
    T* get() const { return obj_; }
    explicit operator bool() const { return obj_ != nullptr; }
};

// ── Object Pool ─────────────────────────────────────────

template <typename T>
class ObjectPool {
    std::vector<std::unique_ptr<T>> all_objects_;   // owns every T ever created
    std::stack<T*> available_;                       // free objects ready for reuse

    std::function<std::unique_ptr<T>()> factory_;    // how to create a new T
    std::function<void(T&)> resetter_;               // how to reset T for reuse

    std::size_t max_size_;                           // 0 = unlimited

public:
    explicit ObjectPool(
        std::function<std::unique_ptr<T>()> factory,
        std::function<void(T&)> resetter = [](T&){},
        std::size_t initial_size = 0,
        std::size_t max_size = 0)
        : factory_(std::move(factory))
        , resetter_(std::move(resetter))
        , max_size_(max_size)
    {
        for (std::size_t i = 0; i < initial_size; ++i) {
            grow_one();
        }
    }

    // Acquire a raw pointer (caller must release)
    T* acquire() {
        if (available_.empty()) {
            if (max_size_ > 0 && all_objects_.size() >= max_size_) {
                throw std::runtime_error("ObjectPool: max size reached");
            }
            grow_one();
        }
        T* obj = available_.top();
        available_.pop();
        return obj;
    }

    // Acquire with RAII handle (automatic release)
    PoolHandle<T> acquire_handle() {
        return PoolHandle<T>(this, acquire());
    }

    // Return an object to the pool
    void release(T* obj) {
        resetter_(*obj);
        available_.push(obj);
    }

    // Stats
    std::size_t total_created() const { return all_objects_.size(); }
    std::size_t available_count() const { return available_.size(); }
    std::size_t in_use() const { return all_objects_.size() - available_.size(); }

private:
    void grow_one() {
        auto obj = factory_();
        available_.push(obj.get());
        all_objects_.push_back(std::move(obj));
    }
};
```

  
  


---

  


## 📘 13.9 Design Walkthrough

**Why `vector<unique_ptr<T>>` + `stack<T*>`?**

```
all_objects_ (owns everything):
  ┌──────────┬──────────┬──────────┬──────────┐
  │ uptr→ T0 │ uptr→ T1 │ uptr→ T2 │ uptr→ T3 │
  └──────────┴──────────┴──────────┴──────────┘

available_ (raw pointers to idle objects):
  ┌────┐
  │ T1 │ ← top (next to be acquired)
  ├────┤
  │ T3 │
  └────┘

in-use (not in available_ stack):
  T0 — some client is using it
  T2 — some client is using it
```

- `all_objects_` provides **ownership** — when the pool is destroyed, all objects are cleaned up via `unique_ptr` destructors
- `available_` is a **free list** of raw pointers — O(1) push/pop
- No object is ever truly destroyed until the pool itself dies

**Why a factory function?**

Different `T` types need different constructor arguments. A factory lambda captures them:

```cpp
auto pool = ObjectPool<DatabaseConn>(
    []() { return std::make_unique<DatabaseConn>("host=db.local port=5432"); },
    [](DatabaseConn& c) { c.rollback(); c.clear_state(); },
    /*initial=*/ 5,
    /*max=*/ 20
);
```

**Why a resetter function?**

Same idea — how to "clean" an object depends on the type. Separating it from the pool makes the pool generic.

  
  


---

---

# PART 4: REAL-WORLD EXAMPLES

---

---

  


## 📘 13.10 Connection Pool

The most classic object pool. Every database library has one:

```cpp
struct Connection {
    int id;
    bool in_transaction = false;
    std::string pending_query;

    Connection(int i) : id(i) {
        std::cout << "  [EXPENSIVE] Connection " << id << " opened (handshake, auth)\n";
    }
    ~Connection() {
        std::cout << "  Connection " << id << " closed\n";
    }

    void execute(const std::string& sql) {
        std::cout << "  Connection " << id << " executing: " << sql << "\n";
    }
    void reset() {
        in_transaction = false;
        pending_query.clear();
    }
};

int next_id = 1;
ObjectPool<Connection> conn_pool(
    [&]() { return std::make_unique<Connection>(next_id++); },
    [](Connection& c) { c.reset(); },
    /*initial=*/ 3,
    /*max=*/ 10
);

// Usage:
{
    auto conn = conn_pool.acquire_handle();
    conn->execute("SELECT * FROM users");
}   // connection returned to pool, not destroyed
```

  
  


---

  


## 📘 13.11 Thread Pool (Conceptual)

A thread pool is conceptually an object pool for **worker threads**:

```
Pool of 4 threads:
  ┌──────────┬──────────┬──────────┬──────────┐
  │ Thread 0 │ Thread 1 │ Thread 2 │ Thread 3 │
  │  idle    │  busy    │  idle    │  busy    │
  └──────────┴──────────┴──────────┴──────────┘

submit(task):
  → find an idle thread → give it the task → thread becomes busy
  → when task completes → thread becomes idle (returned to pool)
```

You won't build a thread pool as `ObjectPool<std::thread>` directly (threads have more complex lifecycle), but the **pattern** is the same: pre-create expensive resources, reuse them.

  
  


---

  


## 📘 13.12 Buffer Pool

Pre-allocate large buffers for I/O or network operations:

```cpp
struct Buffer {
    static constexpr std::size_t SIZE = 4096;
    char data[SIZE];
    std::size_t length = 0;

    void reset() { length = 0; std::memset(data, 0, SIZE); }
};

ObjectPool<Buffer> buf_pool(
    []() { return std::make_unique<Buffer>(); },
    [](Buffer& b) { b.reset(); },
    /*initial=*/ 100
);

// In a network server:
void handle_request(int fd) {
    auto buf = buf_pool.acquire_handle();
    buf->length = read(fd, buf->data, Buffer::SIZE);
    process(buf->data, buf->length);
}   // buffer returned to pool, ready for next request
```

  
  


---

---

# PART 5: THREAD SAFETY

---

---

  


## 📘 13.13 Adding a Mutex

For multi-threaded use, guard `acquire()` and `release()` with a mutex:

```cpp
#include <mutex>

template <typename T>
class ThreadSafeObjectPool {
    ObjectPool<T> pool_;
    std::mutex mtx_;

public:
    template <typename... PoolArgs>
    ThreadSafeObjectPool(PoolArgs&&... args)
        : pool_(std::forward<PoolArgs>(args)...) {}

    T* acquire() {
        std::lock_guard<std::mutex> lock(mtx_);
        return pool_.acquire();
    }

    PoolHandle<T> acquire_handle() {
        std::lock_guard<std::mutex> lock(mtx_);
        return pool_.acquire_handle();
    }

    void release(T* obj) {
        std::lock_guard<std::mutex> lock(mtx_);
        pool_.release(obj);
    }
};
```

This is the simple approach. For high-contention scenarios, consider:

- **Lock-free stack** (Treiber stack from Day 10 bonus) for the available list
- **Thread-local pools** with periodic redistribution
- **Sharded pools** (one sub-pool per thread, steal from others if empty)

  
  


---

  


## 📘 13.14 Thread-Local Pool (Better Scaling)

```
Instead of one shared pool with a lock:

Thread 0: ┌──────────────────┐
          │ local pool: T0,T1│ ← no contention
          └──────────────────┘

Thread 1: ┌──────────────────┐
          │ local pool: T2,T3│ ← no contention
          └──────────────────┘

Thread 2: ┌──────────────────┐
          │ local pool: T4,T5│ ← no contention
          └──────────────────┘
```

Each thread has its own small pool. If a thread's pool is empty, it can **steal** from a global overflow pool (with a lock). This is how jemalloc and tcmalloc handle thread-local caches.

  
  


---

---

# PART 6: EDGE CASES & PITFALLS

---

---

  


## 📘 13.15 Common Mistakes

  


#### Pitfall 1: Using after release

```cpp
auto* conn = pool.acquire();
pool.release(conn);
conn->execute("SELECT 1");   // BUG: conn may already be acquired by another caller
```

Fix: use `PoolHandle` (RAII) so you can't accidentally use a released object.

  


#### Pitfall 2: Forgetting to release

```cpp
void handle(ObjectPool<Buffer>& pool) {
    Buffer* buf = pool.acquire();
    process(buf);
    if (error) return;       // BUG: buf never released — pool "leaks" (object stuck in-use)
    pool.release(buf);
}
```

Fix: `PoolHandle` — destructor releases automatically even on early return or exception.

  


#### Pitfall 3: Pool exhaustion without handling

```cpp
auto* obj = pool.acquire();   // throws if max reached
// If you don't catch, program crashes
```

Decide your strategy: block and wait (with a condition variable), return `nullptr`, throw, or grow.

  


#### Pitfall 4: Stale state leaking between users

```cpp
auto* conn = pool.acquire();
conn->begin_transaction();
// ... crash or early return ...
pool.release(conn);
// Next user gets a connection mid-transaction!
```

Fix: the resetter must handle all state cleanup. For connections: rollback any open transaction, clear prepared statements, reset session variables.

  


#### Pitfall 5: Objects holding external resources

If `T` holds file handles, sockets, or locks, the reset function must release those — otherwise pooled objects accumulate stale resources.

  
  


---

---

# PART 7: BUILD EXERCISE

---

---

  


## 📘 13.16 Exercise: Build `ObjectPool`

Implement the following and test with the driver below:

```cpp
template <typename T>
class ObjectPool {
    // YOUR IMPLEMENTATION:
    // - vector<unique_ptr<T>> for ownership
    // - stack<T*> for available objects
    // - factory function
    // - resetter function
    // - initial_size, max_size

public:
    ObjectPool(
        std::function<std::unique_ptr<T>()> factory,
        std::function<void(T&)> resetter = [](T&){},
        std::size_t initial_size = 0,
        std::size_t max_size = 0);

    T* acquire();
    PoolHandle<T> acquire_handle();
    void release(T* obj);

    std::size_t total_created() const;
    std::size_t available_count() const;
    std::size_t in_use() const;
};

template <typename T>
class PoolHandle {
    // YOUR IMPLEMENTATION:
    // - Move-only RAII handle
    // - operator*, operator->, get()
    // - Releases on destruction
};
```

  


### Test Driver

```cpp
struct ExpensiveObject {
    int id;
    std::string state;

    ExpensiveObject(int i) : id(i), state("initialized") {
        std::cout << "  [CONSTRUCT] ExpensiveObject " << id << " (took 100ms pretend)\n";
    }
    ~ExpensiveObject() {
        std::cout << "  [DESTROY] ExpensiveObject " << id << "\n";
    }
    void do_work(const std::string& s) {
        state = s;
        std::cout << "  Object " << id << " working: " << s << "\n";
    }
    void reset() {
        state.clear();
        std::cout << "  Object " << id << " reset\n";
    }
};

int main() {
    int next_id = 1;

    std::cout << "=== 1. Pool creation with pre-warming ===\n";
    ObjectPool<ExpensiveObject> pool(
        [&]() { return std::make_unique<ExpensiveObject>(next_id++); },
        [](ExpensiveObject& o) { o.reset(); },
        /*initial=*/ 3,
        /*max=*/ 5
    );
    std::cout << "  total=" << pool.total_created()
              << " available=" << pool.available_count()
              << " in_use=" << pool.in_use() << "\n";

    std::cout << "\n=== 2. Acquire and release (manual) ===\n";
    {
        auto* obj = pool.acquire();
        obj->do_work("task A");
        std::cout << "  in_use=" << pool.in_use() << "\n";
        pool.release(obj);
        std::cout << "  in_use=" << pool.in_use() << "\n";
    }

    std::cout << "\n=== 3. Acquire with RAII handle ===\n";
    {
        auto handle = pool.acquire_handle();
        handle->do_work("task B");
        std::cout << "  in_use=" << pool.in_use() << "\n";
    }
    std::cout << "  after scope: in_use=" << pool.in_use() << "\n";

    std::cout << "\n=== 4. Exhaust pool → grow on demand ===\n";
    {
        auto h1 = pool.acquire_handle();
        auto h2 = pool.acquire_handle();
        auto h3 = pool.acquire_handle();
        std::cout << "  3 acquired, total=" << pool.total_created() << "\n";

        auto h4 = pool.acquire_handle();
        std::cout << "  4 acquired, total=" << pool.total_created()
                  << " (grew on demand)\n";

        auto h5 = pool.acquire_handle();
        std::cout << "  5 acquired, total=" << pool.total_created()
                  << " (at max)\n";

        try {
            auto h6 = pool.acquire_handle();   // should fail
        } catch (const std::runtime_error& e) {
            std::cout << "  caught: " << e.what() << "\n";
        }
    }
    std::cout << "  all released: in_use=" << pool.in_use() << "\n";

    std::cout << "\n=== 5. Reuse verification ===\n";
    {
        auto h1 = pool.acquire_handle();
        int* addr1 = &h1->id;
        h1->do_work("first use");
    }
    {
        auto h2 = pool.acquire_handle();
        int* addr2 = &h2->id;
        h2->do_work("second use — same object reused");
        std::cout << "  state after acquire (should be empty): '"
                  << h2->state << "'\n";
    }

    std::cout << "\n=== 6. Move handle ===\n";
    {
        auto h1 = pool.acquire_handle();
        h1->do_work("original");
        auto h2 = std::move(h1);
        std::cout << "  h1 valid: " << (h1 ? "yes" : "no") << "\n";
        std::cout << "  h2 valid: " << (h2 ? "yes" : "no") << "\n";
        h2->do_work("after move");
    }

    std::cout << "\n=== Done — pool destroyed, all objects cleaned up ===\n";
    return 0;
}
```

  


### Compile and run

```bash
g++ -std=c++17 -Wall -Wextra -fsanitize=address -o day13 day13_object_pool.cpp && ./day13
```

  


#### Bonus Challenges

1. **Blocking acquire**: Instead of throwing when the pool is full, add an `acquire_blocking()` that waits on a `std::condition_variable` until an object is released.
2. **Time-to-live**: Destroy objects that have been idle for longer than N seconds. Run a background thread that periodically sweeps the available stack.
3. **Health check**: Before handing out a connection, validate it's still alive (`ping()`). If dead, discard and create a new one.
4. `**shared_ptr` with custom deleter**: Instead of `PoolHandle`, return a `shared_ptr<T>` whose deleter calls `pool.release()`:

```cpp
std::shared_ptr<T> acquire_shared() {
    T* obj = acquire();
    return std::shared_ptr<T>(obj, [this](T* p) { this->release(p); });
}
```

1. **Benchmark**: Compare creating/destroying `ExpensiveObject` per iteration vs reusing from pool for 10,000 iterations.

  
  


---

---

# PART 8: DEEP DIVE — COMMON QUESTIONS

---

---

  


## 📘 13.17 Q&A

  


#### Q1: "Why not just cache a `static` object?"

A single static object can't be used by multiple callers simultaneously. A pool provides **multiple** reusable instances. Also, a static object has global lifetime — a pool has scoped lifetime and controlled size.

  


#### Q2: "Should the pool own the objects or should clients own them?"

The **pool owns** all objects (via `unique_ptr` in `all_objects_`). Clients get a **borrow** (raw pointer or `PoolHandle`). This is the same ownership model as `unique_ptr` vs raw pointer: clear single owner, non-owning borrows.

  


#### Q3: "What if `T` doesn't have a `reset()` method?"

Three options:

- **Default resetter**: `[](T&){}` — do nothing (fine if `T` is stateless or always overwritten before use)
- **Destroy and re-create**: On release, destroy the old object and placement-new a fresh one. More expensive, but guarantees clean state
- **Require a concept**: Use a `Resettable` concept or CRTP base class

  


#### Q4: "Pool vs Flyweight pattern?"


| Pool                                                         | Flyweight                                                      |
| ------------------------------------------------------------ | -------------------------------------------------------------- |
| Objects are **exclusively checked out** (one user at a time) | Objects are **shared** (many users, read-only intrinsic state) |
| Mutable state per user                                       | Immutable shared state + extrinsic state per user              |
| Goal: avoid construction cost                                | Goal: avoid memory duplication                                 |


  


#### Q5: "How do database connection pools handle max connections?"

Typical strategy:

1. **Pre-warm** N connections at startup
2. Grow on demand up to **max** (e.g., 20)
3. If all in use: **block** with a timeout (wait for a release)
4. If timeout expires: throw "connection pool exhausted"
5. **Health check**: periodically ping idle connections, replace dead ones

  


#### Q6: "Can I use `std::shared_ptr` with a custom deleter instead of `PoolHandle`?"

Yes — and it's a common real-world pattern:

```cpp
auto conn = std::shared_ptr<Connection>(
    pool.acquire(),
    [&pool](Connection* c) { pool.release(c); }
);
```

The `shared_ptr`'s deleter calls `release()` instead of `delete`. This integrates with existing code that expects `shared_ptr`. The downside: `shared_ptr` has refcount overhead (control block allocation), while `PoolHandle` is as lightweight as `unique_ptr`.

  
  


---

## Reflection Questions

1. How does an object pool differ from a pool allocator?
2. When is the construction cost high enough to justify a pool?
3. Why is RAII essential for safe pool usage?
4. What state must a resetter clean up for a database connection?
5. When would you prefer a thread-local pool over a mutex-guarded pool?

---

## Interview Questions

1. "What is the Object Pool pattern? When would you use it?"
2. "How does an object pool differ from a memory pool allocator?"
3. "Design a database connection pool. What are the key decisions?"
4. "How would you make an object pool thread-safe?"
5. "What happens if a client forgets to return an object to the pool?"
6. "How do you handle stale state in pooled objects?"
7. "Implement an RAII handle for automatic pool release."
8. "Compare: `PoolHandle` vs `shared_ptr` with custom deleter for pool return."

---

**Next**: Day 14 — Weekly Review →