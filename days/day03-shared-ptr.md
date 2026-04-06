# Day 3: `shared_ptr` & `weak_ptr` Deep Dive

[← Back to Study Plan](../lld-study-plan.md) | [← Day 2](day02-unique-ptr.md)

> **Time**: ~2 hours  
> **Goal**: Understand shared ownership, the control block, weak pointers, and implement `SharedPtr<T>` + `WeakPtr<T>` with a single shared control block.

---
---

# PART 1: THEORY (35 min)

---

<br>

<h2 style="color: #2980B9;">📘 3.1 When One Owner Is Not Enough</h2>

`unique_ptr` models **exclusive** ownership: exactly one smart pointer deletes the object.

Sometimes several parts of a program need to **share** responsibility for the same object’s lifetime:

- A cache and a request handler both hold a handle to the same large buffer
- Multiple observers reference the same subject
- A graph of nodes with back-references

`std::shared_ptr<T>` uses **reference counting**: the last `shared_ptr` that releases the object runs the destructor.

```cpp
auto p = std::make_shared<Widget>(42);
{
    auto q = p;              // two owners — refcount = 2
    use(*q);
}                            // q destroyed — refcount = 1
// Widget still alive — p still owns it
```

**Rule of thumb**: Prefer `unique_ptr` when you can name a single owner. Reach for `shared_ptr` when shared lifetime is a real requirement, not convenience.

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 3.2 The Control Block (Mental Model)</h2>

A `shared_ptr` is **not** just a raw pointer. It points at a **control block** (on the heap) that stores at least:

| Field | Role |
|-------|------|
| **Strong reference count** | Number of `shared_ptr` instances sharing ownership. When it hits 0, the **managed object** is destroyed. |
| **Weak reference count** | Number of `weak_ptr` instances (and sometimes used to delay control-block deletion — see below). |
| **Deleter** | Type-erased callable to destroy the object (default: `delete`). |
| **Allocator** (optional) | Where the control block and possibly the object were allocated. |

The **managed object** may be:

- **Separate allocation**: `std::shared_ptr<T>(new T)` → one allocation for `T`, one for the control block (often **two** heap allocations total unless optimized).
- **Combined**: `std::make_shared<T>(...)` → typically **one** allocation for object + control block together (faster, better cache locality; object and block freed together).

```
shared_ptr sp ──►  [ Control block ]     +     [ T object ]   (layout varies)
                    strong: 2
                    weak: 1
                    deleter, vtable ptrs...
```

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 3.3 Strong vs Weak Count — Two Different “Deaths”</h2>

1. **When strong count → 0**: The **managed object** is destroyed (destructor runs).

2. **When strong count → 0 AND weak count → 0**: The **control block** is destroyed and memory reclaimed.

So `weak_ptr` can outlive the object: it keeps the **control block** alive (so you can ask `expired()` / `lock()`), but it does **not** keep the `T` alive.

```cpp
std::weak_ptr<Widget> w;
{
    auto p = std::make_shared<Widget>();
    w = p;
}   // p gone — strong count 0 → Widget destroyed
    // but weak_ptr w still valid as a handle — control block lives until w is destroyed or reset
auto locked = w.lock();   // returns empty shared_ptr if object is gone
```

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 3.4 <code>weak_ptr</code> — Non-Owning Observer</h2>

| Operation | Meaning |
|-----------|---------|
| `weak_ptr(w)` / `operator=` from `shared_ptr` | Associate with same control block; does **not** increment strong count |
| `lock()` | If object alive, returns `shared_ptr` (increments strong count); else empty `shared_ptr` |
| `expired()` | `true` if strong count is 0 |
| `use_count()` | Current strong count (approximate / for debugging) |

**Classic use**: break **cycles** between `shared_ptr`s (e.g. parent ↔ child). One direction should be `weak_ptr` so strong counts can drop to zero.

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 3.5 Aliasing Constructor</h2>

You can build a `shared_ptr` that shares ownership with an existing `shared_ptr` but points at a **different address** (e.g. a member or base subobject):

```cpp
struct Outer { Inner inner; };
auto outer = std::make_shared<Outer>();
std::shared_ptr<Inner> inner_alias(outer, &outer->inner);
// Refcount tied to same control block as `outer`; deleting happens when last alias dies.
```

Useful for APIs that need a `shared_ptr` to a member while lifetime stays tied to the whole `Outer`.

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 3.6 <code>make_shared</code> vs <code>shared_ptr(new T)</code></h2>

| | `make_shared<T>(args...)` | `shared_ptr<T>(new T(args...))` |
|---|---------------------------|----------------------------------|
| Allocations | Usually **one** (object + control block) | Often **two** |
| Exception safety | Single allocation path | `new` then block — extra care pre-C++11 |
| `weak_ptr` memory | Object stays allocated until all weak gone if using combined storage | Same idea when combined |

**Caveat**: If you need a custom deleter that must run in a special way, or array `shared_ptr`, APIs differ — read the standard library docs for your version.

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 3.7 Thread Safety</h2>

- **Control block updates** (increment/decrement strong/weak counts) are **atomic** — safe to copy/move `shared_ptr` between threads that use the same pointer.
- **The pointed-to object `T`** is **not** automatically thread-safe. You still need mutexes or atomics for data races on `*p`.

So: “`shared_ptr` is thread-safe” refers to **reference counting**, not to `T`.

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 3.8 Interface Sketch (<code>std::shared_ptr</code>)</h2>

| Operation | Behavior |
|-----------|----------|
| `shared_ptr()` | Empty |
| `shared_ptr(T*)` | Owns `T` (prefer `make_shared`) |
| Copy ctor / copy assign | Share ownership — strong++ |
| Move ctor / move assign | Transfer — no refcount change for “new” owner semantics |
| `reset()` | Release — strong-- |
| `get()`, `*`, `->` | Like raw pointer |
| `use_count()` | Strong count (debugging) |
| `unique()` | `use_count() == 1` |

`weak_ptr`: `lock()`, `expired()`, `reset()`, `owner_before` for ordering.

<br><br>

---
---

# PART 2: BUILD EXERCISE (55 min)

---

<br>

## Exercise: `SharedPtr<T>` + `WeakPtr<T>` with One Control Block

Implement a **minimal** teaching version (not libc++-grade) with:

- Control block: `strong_count`, `weak_count`, `T* ptr` (or type-erased deleter for bonus)
- `SharedPtr`: copy/move, destructor, `get`, `*`, `->`, `reset`, `use_count`
- `WeakPtr`: construct from `SharedPtr`, `lock()`, `expired()`
- **Self-assignment** and **empty** states safe
- Optional: `make_shared` analogue that allocates `T` + block in one go

<br>

### Design notes

1. **Where to put refcount**: Heap-allocated `ControlBlock` shared by all `SharedPtr`/`WeakPtr` aliases.

2. **Destructor rule**: On last `SharedPtr` release (`strong` → 0): `delete ptr` (or call deleter). If `weak` also 0, `delete control_block`.

3. **When last `SharedPtr` dies but weak > 0**: Destroy `T`, but **keep** control block until last `WeakPtr` dies.

4. **Circular dependency**: If `A` holds `shared_ptr<B>` and `B` holds `shared_ptr<A>`, counts never reach 0 — use `weak_ptr` on one side.

<br>

### Starter skeleton

```cpp
#include <iostream>
#include <utility>
#include <cstddef>

template <typename T>
class SharedPtr;

template <typename T>
class WeakPtr;

template <typename T>
class ControlBlock {
public:
    T* ptr;
    size_t strong;
    size_t weak;

    explicit ControlBlock(T* p) : ptr(p), strong(1), weak(0) {}

    void add_strong() { ++strong; }
    void release_strong();  // implement: maybe delete T, maybe delete block
    void add_weak() { ++weak; }
    void release_weak();    // implement: maybe delete block
};

template <typename T>
class SharedPtr {
    T* ptr_ = nullptr;
    ControlBlock<T>* ctrl_ = nullptr;

    void release() {
        if (!ctrl_) return;
        // TODO: decrement strong, destroy T if 0, destroy block if both 0
    }

public:
    SharedPtr() = default;

    explicit SharedPtr(T* raw) : ptr_(raw) {
        if (raw) ctrl_ = new ControlBlock<T>(raw);
    }

    ~SharedPtr() { release(); }

    SharedPtr(const SharedPtr& other);
    SharedPtr& operator=(const SharedPtr& other);
    SharedPtr(SharedPtr&& other) noexcept;
    SharedPtr& operator=(SharedPtr&& other) noexcept;

    T* get() const { return ptr_; }
    T& operator*() const { return *ptr_; }
    T* operator->() const { return ptr_; }

    size_t use_count() const { return ctrl_ ? ctrl_->strong : 0; }

    void reset(T* raw = nullptr);

    // Friend or accessor for WeakPtr implementation
    ControlBlock<T>* control_block() const { return ctrl_; }
};

template <typename T>
class WeakPtr {
    ControlBlock<T>* ctrl_ = nullptr;

public:
    WeakPtr() = default;
    WeakPtr(const SharedPtr<T>& sp);
    WeakPtr(const WeakPtr& other);
    WeakPtr& operator=(const WeakPtr& other);
    ~WeakPtr();

    bool expired() const;
    SharedPtr<T> lock() const;
};
```

Implement `ControlBlock<T>::release_strong` / `release_weak` **out of line** after `SharedPtr` / `WeakPtr` are complete, or use free functions to avoid circular dependencies.

<br>

### Test ideas

- Two `SharedPtr`s to same object — `use_count() == 2`, one goes out of scope → `1`.
- `WeakPtr` — after all `SharedPtr`s gone, `expired() == true`, `lock()` empty.
- Copy assignment `a = b` — old resource of `a` released correctly.
- Self-assignment `p = p` — no double-free.

```bash
g++ -std=c++17 -Wall -Wextra -fsanitize=address -o day03 day03_shared_ptr.cpp && ./day03
```

<br><br>

---
---

# PART 3: DEEP DIVE CHALLENGES (20 min)

---

<br>

## Challenge 1: `enable_shared_from_this`

Read `std::enable_shared_from_this<T>`: why `shared_from_this()` is invalid unless `this` is already managed by a `shared_ptr`.

## Challenge 2: Aliasing

Implement a minimal aliasing constructor:

```cpp
template <typename U>
SharedPtr(SharedPtr<T> owner, U* alias);
```

## Challenge 3: Thread-safe counts

Replace `size_t` with `std::atomic<size_t>` for `strong`/`weak` and reason about `memory_order` (Day 4 preview).

## Challenge 4: `make_shared` single allocation

Allocate one blob: `[ControlBlock][T]` with placement new for `T`.

<br><br>

---
---

# PART 4: DEEP DIVE Q&A

---

<br>

<h2 style="color: #2980B9;">📘 Q1: Factory returns <code>unique_ptr</code> — is the returned <code>unique_ptr</code> “destroyed”?</h2>

Yes: the **temporary** `unique_ptr` in the callee is destroyed **after** the return value is initialized — ownership is **moved** into the caller’s `unique_ptr` (or prvalue elision applies). The **pointed-to object** is not destroyed unless you returned null or moved from an empty pointer.

```cpp
std::unique_ptr<Widget> make_widget() {
    return std::make_unique<Widget>();  // prvalue → caller’s unique_ptr
}
// No leak: exactly one unique_ptr owns the Widget at any time.
```

So “destroyed” applies to the small `unique_ptr` object, not necessarily the heap `Widget`.

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 Q2: If <code>operator*</code> is <code>const</code>, can I still assign through the reference?</h2>

For a **const** smart pointer, `operator*() const` typically returns `T&` (not `const T&`) so you can still write `*p = value` — the **pointer object** is const, not the **pointed-to object**. That matches `std::unique_ptr` / `std::shared_ptr`.

If you want to prevent modifying `T` through a const smart pointer, you’d return `const T&` from `operator*() const` (non-standard for `shared_ptr`, but valid for a teaching type).

```cpp
const std::shared_ptr<int> p = std::make_shared<int>(1);
*p = 42;   // OK with std::shared_ptr — *p is int&
```

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 Q3: Why doesn’t <code>weak_ptr</code> keep the object alive?</h2>

By design: it is a **non-owning** handle. If it incremented the strong count, it would become another `shared_ptr` in disguise and you couldn’t break cycles. Keeping only the **control block** alive lets you query `expired()` and `lock()` without extending `T`’s lifetime.

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 Q4: Two control blocks for the same raw pointer — disaster</h2>

```cpp
int* raw = new int(42);
std::shared_ptr<int> a(raw);
std::shared_ptr<int> b(raw);   // UB: second control block — double delete
```

Always: second `shared_ptr` from first: `shared_ptr<int> b(a)` or `b = a`, or `shared_ptr<int>(a)` after `a` already owns `raw`.

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 Q5: <code>shared_ptr</code> to <code>this</code> — why is it dangerous?</h2>

Creating `shared_ptr<T>(this)` from inside a member function creates a **new** control block unrelated to existing owners — double free. Use `enable_shared_from_this` when an object must hand out `shared_ptr`s to itself after already owned by `shared_ptr`.

<br><br>

---
---

# PART 5: REFLECT & NOTES (10 min)

---

<br>

1. **When do you choose `shared_ptr` over `unique_ptr`?**  
2. **What lives in the control block vs the managed object?**  
3. **How does `weak_ptr` break reference cycles?**  
4. **What is thread-safe about `shared_ptr`, and what is not?**  
5. **Why prefer `make_shared` when possible?**

<br><br>

---
---

# INTERVIEW QUESTIONS TO PRACTICE

---

<br>

1. How does `shared_ptr` know when to delete the object?
2. What is the difference between strong and weak counts?
3. What is `weak_ptr` for? Give a cycle example.
4. Is `shared_ptr` thread-safe? Explain precisely.
5. What is the aliasing constructor?
6. Why is `shared_from_this` needed?
7. Implement a simplified `shared_ptr` control block.

<br><br>

---
---

# REFERENCE SOLUTION

---

<br>

<details>
<summary>Click to expand a minimal SharedPtr / WeakPtr sketch</summary>

<br>

```cpp
#include <utility>
#include <cstddef>

template <typename T> class SharedPtr;
template <typename T> class WeakPtr;

template <typename T>
class ControlBlock {
public:
    T* ptr;
    size_t strong;
    size_t weak;

    explicit ControlBlock(T* p) : ptr(p), strong(1), weak(0) {}
};

template <typename T>
class SharedPtr {
    friend class WeakPtr<T>;

    T* ptr_ = nullptr;
    ControlBlock<T>* ctrl_ = nullptr;

    // Used by WeakPtr::lock — ctrl_ non-null and strong > 0
    SharedPtr(ControlBlock<T>* c, T* p) : ptr_(p), ctrl_(c) {
        if (ctrl_) ++ctrl_->strong;
    }

    void release() {
        if (!ctrl_) return;
        --ctrl_->strong;
        if (ctrl_->strong == 0) {
            delete ctrl_->ptr;
            ctrl_->ptr = nullptr;
        }
        if (ctrl_->strong == 0 && ctrl_->weak == 0) {
            delete ctrl_;
        }
        ptr_ = nullptr;
        ctrl_ = nullptr;
    }

public:
    SharedPtr() = default;

    explicit SharedPtr(T* raw) : ptr_(raw) {
        if (raw) ctrl_ = new ControlBlock<T>(raw);
    }

    ~SharedPtr() { release(); }

    SharedPtr(const SharedPtr& o) : ptr_(o.ptr_), ctrl_(o.ctrl_) {
        if (ctrl_) ++ctrl_->strong;
    }

    SharedPtr& operator=(const SharedPtr& o) {
        if (this == &o) return *this;
        release();
        ptr_ = o.ptr_;
        ctrl_ = o.ctrl_;
        if (ctrl_) ++ctrl_->strong;
        return *this;
    }

    SharedPtr(SharedPtr&& o) noexcept : ptr_(o.ptr_), ctrl_(o.ctrl_) {
        o.ptr_ = nullptr;
        o.ctrl_ = nullptr;
    }

    SharedPtr& operator=(SharedPtr&& o) noexcept {
        if (this == &o) return *this;
        release();
        ptr_ = o.ptr_;
        ctrl_ = o.ctrl_;
        o.ptr_ = nullptr;
        o.ctrl_ = nullptr;
        return *this;
    }

    T* get() const { return ptr_; }
    T& operator*() const { return *ptr_; }
    T* operator->() const { return ptr_; }
    size_t use_count() const { return ctrl_ ? ctrl_->strong : 0; }

    void reset(T* raw = nullptr) {
        release();
        if (raw) {
            ptr_ = raw;
            ctrl_ = new ControlBlock<T>(raw);
        }
    }
};

template <typename T>
class WeakPtr {
    ControlBlock<T>* ctrl_ = nullptr;

    void release_weak() {
        if (!ctrl_) return;
        --ctrl_->weak;
        if (ctrl_->strong == 0 && ctrl_->weak == 0) delete ctrl_;
        ctrl_ = nullptr;
    }

public:
    WeakPtr() = default;

    WeakPtr(const SharedPtr<T>& sp) : ctrl_(sp.ctrl_) {
        if (ctrl_) ++ctrl_->weak;
    }

    WeakPtr(const WeakPtr& o) : ctrl_(o.ctrl_) {
        if (ctrl_) ++ctrl_->weak;
    }

    WeakPtr& operator=(const WeakPtr& o) {
        if (this == &o) return *this;
        release_weak();
        ctrl_ = o.ctrl_;
        if (ctrl_) ++ctrl_->weak;
        return *this;
    }

    ~WeakPtr() { release_weak(); }

    bool expired() const {
        return !ctrl_ || ctrl_->strong == 0;
    }

    SharedPtr<T> lock() const {
        if (expired()) return SharedPtr<T>();
        return SharedPtr<T>(ctrl_, ctrl_->ptr);
    }
};
```

**Details to add in your own build:** `WeakPtr` move operations, `WeakPtr& operator=(SharedPtr<T> const&)`, `reset()`, and `owner_before`. Use `friend` carefully or keep `control_block()` accessors — here `WeakPtr` reads `SharedPtr`’s private `ctrl_` via the shown constructor path only.

</details>

<br><br>

---

**End of Day 3**
