# Day 2: `unique_ptr` Deep Dive

[← Back to Study Plan](../lld-study-plan.md) | [← Day 1](day01-move-semantics-rule-of-5.md)

> **Time**: ~2 hours
> **Goal**: Understand exclusive ownership semantics, how `unique_ptr` works internally, and implement one from scratch with custom deleter support.

---
---

# PART 1: THEORY (30 min)

---

<br>

<h2 style="color: #2980B9;">📘 2.1 What Problem Does <code>unique_ptr</code> Solve?</h2>

Raw pointers have no ownership semantics — the compiler doesn't know who should `delete` them:

```cpp
void bad() {
    int* p = new int(42);
    if (someCondition()) return;   // LEAK: p is never deleted
    delete p;
}

void also_bad() {
    int* p = new int(42);
    doSomething(p);   // does doSomething delete p? who knows?
    delete p;         // maybe double delete?
}
```

`unique_ptr` solves this with **RAII** — it wraps a raw pointer and deletes it automatically when the `unique_ptr` goes out of scope. There is exactly **one owner** at any time.

```cpp
void good() {
    auto p = std::make_unique<int>(42);
    if (someCondition()) return;   // p's destructor runs → deletes the int
    // no explicit delete needed
}
```

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 2.2 Exclusive Ownership — The Core Rule</h2>

`unique_ptr` enforces a simple invariant: **exactly one `unique_ptr` owns the resource at any time**.

This means:
- **No copying** — copying would create two owners → double delete
- **Move only** — ownership transfers from source to destination
- After a move, the source is `nullptr`

```cpp
auto a = std::make_unique<int>(42);
// auto b = a;                  // ✗ COMPILE ERROR: copy constructor is deleted
auto b = std::move(a);          // ✓ ownership transferred to b
// a is now nullptr
```

<br>

#### How this maps to the Rule of 5

```cpp
// Inside unique_ptr (conceptually):
unique_ptr(const unique_ptr&) = delete;              // no copy
unique_ptr& operator=(const unique_ptr&) = delete;   // no copy assign

unique_ptr(unique_ptr&& other) noexcept;             // move: steal the pointer
unique_ptr& operator=(unique_ptr&& other) noexcept;  // move assign: steal + delete old
```

This is the first real pattern you'll see where **copy is deleted and only move is allowed**. Your `String` class from Day 1 allowed both — `unique_ptr` is move-only.

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 2.3 The Interface of <code>std::unique_ptr</code></h2>

Key operations you'll need to implement:

| Method | What it does |
|--------|-------------|
| `unique_ptr(T* ptr)` | Take ownership of raw pointer |
| `~unique_ptr()` | Delete the owned object |
| `unique_ptr(unique_ptr&&)` | Move constructor — steal ownership |
| `operator=(unique_ptr&&)` | Move assignment — steal + delete old |
| `T* get()` | Return raw pointer without releasing ownership |
| `T& operator*()` | Dereference — access the object |
| `T* operator->()` | Arrow — access object members |
| `T* release()` | Give up ownership, return raw pointer (caller must delete) |
| `void reset(T* ptr)` | Delete current, take ownership of new pointer |
| `operator bool()` | Check if non-null |
| `swap()` | Exchange owned pointers |

<br>

#### `release()` vs `reset()` — commonly confused

```cpp
auto p = std::make_unique<int>(42);

int* raw = p.release();   // p is now nullptr, caller MUST delete raw
delete raw;

auto q = std::make_unique<int>(10);
q.reset(new int(20));     // deletes the 10, now owns the 20
q.reset();                // deletes the 20, now owns nothing (nullptr)
```

`release()` = "I don't want to own this anymore, you deal with it"
`reset()` = "Delete what I have, optionally take something new"

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 2.4 Custom Deleters</h2>

By default, `unique_ptr` calls `delete` (or `delete[]`). But what if your resource isn't heap memory?

```cpp
// File handle — needs close(), not delete
FILE* f = fopen("data.txt", "r");

// Custom deleter as a lambda
auto file = std::unique_ptr<FILE, decltype(&fclose)>(f, &fclose);
// when 'file' goes out of scope → fclose(f) is called automatically

// Custom deleter as a functor
struct FileCloser {
    void operator()(FILE* f) const { 
        if (f) fclose(f); 
    }
};
std::unique_ptr<FILE, FileCloser> file2(fopen("data.txt", "r"));
```

Other real-world examples:
- `close()` for file descriptors
- `free()` for C-allocated memory (`malloc`)
- `SDL_DestroyWindow()` for SDL resources
- Custom pool deallocation

<br>

#### How the deleter is stored

This is an important implementation detail:

- **`std::unique_ptr`** stores the deleter **inside the object** (as a member). If the deleter is a stateless functor or lambda, the compiler uses **Empty Base Optimization (EBO)** so it takes **zero extra bytes**.
- **`std::shared_ptr`** stores the deleter in the **control block** on the heap — the deleter type is **type-erased**.

This means `unique_ptr<T>` with a default deleter is exactly the same size as a raw pointer (8 bytes on 64-bit). This is a **zero-cost abstraction**.

```cpp
static_assert(sizeof(std::unique_ptr<int>) == sizeof(int*));   // passes!
```

But if you use a function pointer as deleter, it adds 8 bytes:

```cpp
// Functor deleter (stateless) → EBO → same size as raw pointer
static_assert(sizeof(std::unique_ptr<FILE, FileCloser>) == sizeof(FILE*));

// Function pointer deleter → adds 8 bytes
static_assert(sizeof(std::unique_ptr<FILE, decltype(&fclose)>) == sizeof(FILE*) + sizeof(void*));
```

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 2.5 <code>make_unique</code> — Why It Exists</h2>

```cpp
// Option 1: Direct construction
std::unique_ptr<Widget> p(new Widget(arg1, arg2));

// Option 2: make_unique (preferred)
auto p = std::make_unique<Widget>(arg1, arg2);
```

Why prefer `make_unique`?

**1. Exception safety (pre-C++17):**

```cpp
process(std::unique_ptr<A>(new A), std::unique_ptr<B>(new B));
```

Pre-C++17, the compiler could evaluate in this order:
1. `new A`
2. `new B` ← if this throws, the `A` is leaked (no unique_ptr owns it yet)
3. Construct `unique_ptr<A>`
4. Construct `unique_ptr<B>`

With `make_unique`, allocation and construction of the `unique_ptr` are a single step — no leak window.

**2. No `new` keyword** — follows the modern C++ guideline of "no raw `new`/`delete` in application code."

**3. Avoids repeating the type:**

```cpp
std::unique_ptr<VeryLongTypeName> p(new VeryLongTypeName(args));  // type written twice
auto p = std::make_unique<VeryLongTypeName>(args);                 // type written once
```

<br>

#### When you CAN'T use `make_unique`

- Custom deleter — `make_unique` always uses `delete`
- Array with non-default initialization — `make_unique<int[]>(5)` zero-initializes, you can't pass values per element
- Adopting an existing pointer from a C API

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 2.6 <code>unique_ptr</code> and Polymorphism</h2>

`unique_ptr` works naturally with inheritance:

```cpp
class Animal {
public:
    virtual ~Animal() = default;
    virtual void speak() = 0;
};

class Dog : public Animal {
public:
    void speak() override { std::cout << "Woof\n"; }
};

// Implicit conversion: unique_ptr<Derived> → unique_ptr<Base>
std::unique_ptr<Animal> pet = std::make_unique<Dog>();
pet->speak();  // "Woof"
```

The key requirement: **base class must have a virtual destructor**. Otherwise `unique_ptr<Animal>` would call `~Animal()` instead of `~Dog()`, potentially leaking derived class resources.

<br>

#### Why does `unique_ptr<Derived>` implicitly convert to `unique_ptr<Base>`?

The move constructor has a **templated overload**:

```cpp
template <typename U>
unique_ptr(unique_ptr<U>&& other)  // accepts unique_ptr of any compatible type
    requires std::convertible_to<U*, T*>;  // only if U* converts to T* (i.e., U derives from T)
```

This is how the conversion works — it's a move from `unique_ptr<Dog>` into `unique_ptr<Animal>`, enabled only when `Dog*` is convertible to `Animal*`.

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 2.7 Common Patterns & Pitfalls</h2>

<br>

#### Pattern: Factory functions returning `unique_ptr`

```cpp
std::unique_ptr<Shape> createShape(const std::string& type) {
    if (type == "circle") return std::make_unique<Circle>();
    if (type == "square") return std::make_unique<Square>();
    return nullptr;
}
```

This is the **preferred way** to write factory functions — the caller gets clear ownership.

<br>

#### Pattern: Passing `unique_ptr` to functions

```cpp
// Transfer ownership INTO the function (sink)
void consume(std::unique_ptr<Widget> w);     // takes ownership, deletes when done

// Just use the object (most common — don't pass unique_ptr at all)
void use(Widget& w);                          // borrow, no ownership transfer
void use(const Widget& w);                    // borrow read-only

// Might or might not take ownership (rare)
void maybeConsume(std::unique_ptr<Widget>& w); // can reset/release inside
```

**Rule of thumb**: If a function just *uses* the object, pass a reference/pointer. Only pass `unique_ptr` when you're transferring ownership.

<br>

#### Pitfall: Dangling raw pointer after move

```cpp
auto p = std::make_unique<int>(42);
int* raw = p.get();          // raw points to the int
auto q = std::move(p);      // p is now nullptr
std::cout << *raw;           // STILL VALID — q owns it, raw still points to it
q.reset();                   // NOW raw is dangling — the int is deleted
```

`get()` gives you a non-owning view. It doesn't track moves or deletions.

<br>

#### Pitfall: Creating `unique_ptr` from the same raw pointer twice

```cpp
int* raw = new int(42);
std::unique_ptr<int> a(raw);
std::unique_ptr<int> b(raw);   // BUG: both think they own raw → double delete
```

<br><br>

---
---

# PART 2: BUILD EXERCISE (60 min)

---

<br>

## Exercise: Implement `UniquePtr<T>` from Scratch

Build a simplified `unique_ptr` that supports:
- Construction from raw pointer
- Move construction and move assignment
- Copy is deleted
- `get()`, `release()`, `reset()`, `operator*`, `operator->`, `operator bool`
- Custom deleter via template parameter

<br>

### Starter skeleton:

```cpp
#include <iostream>
#include <utility>
#include <cstdio>

// Default deleter — just calls delete
template <typename T>
struct DefaultDeleter {
    void operator()(T* ptr) const {
        std::cout << "  [DefaultDeleter] deleting ptr\n";
        delete ptr;
    }
};

template <typename T, typename Deleter = DefaultDeleter<T>>
class UniquePtr {
private:
    T* ptr_;
    Deleter deleter_;

public:
    // --- Constructor from raw pointer ---
    explicit UniquePtr(T* ptr = nullptr, Deleter d = Deleter())
        : ptr_(ptr), deleter_(d)
    {
        std::cout << "  [ctor] UniquePtr created\n";
    }

    // --- Destructor ---
    // YOUR IMPLEMENTATION HERE
    // Should call deleter_(ptr_) if ptr_ is not null

    // --- Copy: DELETE both ---
    // YOUR IMPLEMENTATION HERE

    // --- Move constructor ---
    // YOUR IMPLEMENTATION HERE
    // Steal ptr_ from other, set other.ptr_ to nullptr

    // --- Move assignment ---
    // YOUR IMPLEMENTATION HERE
    // Delete current resource, steal from other

    // --- get() ---
    // YOUR IMPLEMENTATION HERE

    // --- operator* and operator-> ---
    // YOUR IMPLEMENTATION HERE

    // --- operator bool ---
    // YOUR IMPLEMENTATION HERE

    // --- release() ---
    // Give up ownership, return raw pointer
    // YOUR IMPLEMENTATION HERE

    // --- reset() ---
    // Delete current, optionally take new pointer
    // YOUR IMPLEMENTATION HERE

    // --- swap ---
    // YOUR IMPLEMENTATION HERE
};
```

<br>

### Test driver:

```cpp
// Test helper class
struct Widget {
    int id;
    Widget(int i) : id(i) { std::cout << "  Widget(" << id << ") constructed\n"; }
    ~Widget() { std::cout << "  Widget(" << id << ") destroyed\n"; }
    void greet() { std::cout << "  Hello from Widget " << id << "\n"; }
};

int main() {
    std::cout << "=== 1. Basic construction & destruction ===\n";
    {
        UniquePtr<Widget> p(new Widget(1));
        p->greet();
        std::cout << "  get() = " << p.get() << "\n";
    } // Widget(1) destroyed here

    std::cout << "\n=== 2. Move construction ===\n";
    {
        UniquePtr<Widget> a(new Widget(2));
        UniquePtr<Widget> b = std::move(a);
        std::cout << "  a is null: " << (a.get() == nullptr) << "\n";
        b->greet();
    }

    std::cout << "\n=== 3. Move assignment ===\n";
    {
        UniquePtr<Widget> a(new Widget(3));
        UniquePtr<Widget> b(new Widget(4));
        std::cout << "  -- assigning a = move(b) --\n";
        a = std::move(b);  // Widget(3) should be destroyed here
        a->greet();        // should print Widget 4
    }

    std::cout << "\n=== 4. release() ===\n";
    {
        UniquePtr<Widget> p(new Widget(5));
        Widget* raw = p.release();
        std::cout << "  p is null after release: " << (p.get() == nullptr) << "\n";
        raw->greet();
        delete raw;  // manual cleanup since we released
    }

    std::cout << "\n=== 5. reset() ===\n";
    {
        UniquePtr<Widget> p(new Widget(6));
        std::cout << "  -- reset to new Widget --\n";
        p.reset(new Widget(7));   // Widget(6) destroyed
        p->greet();               // Widget 7
        std::cout << "  -- reset to nullptr --\n";
        p.reset();                // Widget(7) destroyed
        std::cout << "  p is null: " << (p.get() == nullptr) << "\n";
    }

    std::cout << "\n=== 6. operator bool ===\n";
    {
        UniquePtr<Widget> a(new Widget(8));
        UniquePtr<Widget> b;
        std::cout << "  a is truthy: " << (a ? "yes" : "no") << "\n";
        std::cout << "  b is truthy: " << (b ? "yes" : "no") << "\n";
    }

    std::cout << "\n=== 7. Copy is deleted (uncomment to verify) ===\n";
    // UniquePtr<Widget> a(new Widget(9));
    // UniquePtr<Widget> b = a;           // should NOT compile
    // UniquePtr<Widget> c;
    // c = a;                             // should NOT compile
    std::cout << "  (copy lines are commented out — uncomment to verify compile error)\n";

    std::cout << "\n=== 8. Custom deleter ===\n";
    {
        struct FileCloser {
            void operator()(FILE* f) const {
                std::cout << "  [FileCloser] closing file\n";
                if (f) fclose(f);
            }
        };
        UniquePtr<FILE, FileCloser> file(fopen("/dev/null", "r"));
        std::cout << "  file is valid: " << (file ? "yes" : "no") << "\n";
    } // FileCloser called here

    std::cout << "\n=== 9. Self-move-assignment safety ===\n";
    {
        UniquePtr<Widget> p(new Widget(10));
        p = std::move(p);   // must not crash or double-delete
        std::cout << "  survived self-move\n";
    }

    std::cout << "\n=== Done ===\n";
    return 0;
}
```

<br>

### Compile and run:

```bash
g++ -std=c++17 -Wall -Wextra -Wpedantic -fsanitize=address -o day02 day02_unique_ptr.cpp && ./day02
```

<br>

### Expected output pattern:

```
=== 1. Basic construction & destruction ===
  Widget(1) constructed
  [ctor] UniquePtr created
  Hello from Widget 1
  get() = 0x...
  Widget(1) destroyed
  [DefaultDeleter] deleting ptr

=== 2. Move construction ===
  Widget(2) constructed
  [ctor] UniquePtr created
  a is null: 1
  Hello from Widget 2
  Widget(2) destroyed

=== 3. Move assignment ===
  Widget(3) constructed
  Widget(4) constructed
  -- assigning a = move(b) --
  Widget(3) destroyed          ← old resource deleted before steal
  Hello from Widget 4
  Widget(4) destroyed
...
```

<br><br>

---
---

# PART 3: DEEP DIVE CHALLENGES (20 min)

---

<br>

## Challenge 1: Verify zero-cost abstraction

Check that your `UniquePtr<T>` with default deleter is the same size as a raw pointer:

```cpp
static_assert(sizeof(UniquePtr<int>) == sizeof(int*),
    "UniquePtr should be same size as raw pointer with stateless deleter");
```

If this fails, you're storing the deleter as a regular member. To fix it, you need **Empty Base Optimization (EBO)** — inherit from the deleter instead of storing it as a member:

```cpp
template <typename T, typename Deleter = DefaultDeleter<T>>
class UniquePtr : private Deleter {   // EBO: stateless Deleter takes 0 bytes
    T* ptr_;
    // ...
    void destroy() { Deleter::operator()(ptr_); }  // call deleter via base class
};
```

Think about: why does inheriting from an empty class cost 0 bytes, but having it as a member costs at least 1 byte?

<br>

## Challenge 2: Add `make_unique`

Implement a standalone `make_unique` function:

```cpp
template <typename T, typename... Args>
UniquePtr<T> make_unique(Args&&... args) {
    return UniquePtr<T>(new T(std::forward<Args>(args)...));
}
```

This uses **variadic templates** and **perfect forwarding** — two patterns you'll use heavily in systems C++. Think about: why `std::forward` and not `std::move`?

<br>

## Challenge 3: Templated converting constructor

Allow `UniquePtr<Derived>` to convert to `UniquePtr<Base>`:

```cpp
template <typename U, typename UDeleter>
UniquePtr(UniquePtr<U, UDeleter>&& other) noexcept
    requires std::convertible_to<U*, T*>;
```

Test with a base/derived class hierarchy.

<br>

## Challenge 4: Array specialization

`std::unique_ptr<int[]>` uses `delete[]` instead of `delete`. Implement a partial specialization:

```cpp
template <typename T>
class UniquePtr<T[]> {
    // uses delete[] in destructor
    // provides operator[] instead of operator* and operator->
};
```

<br><br>

---
---

# PART 4: DEEP DIVE Q&A

---

<br>

<h2 style="color: #2980B9;">📘 Q1: Why is the constructor <code>explicit</code>?</h2>

```cpp
explicit UniquePtr(T* ptr = nullptr);
```

Without `explicit`, implicit conversions would be allowed:

```cpp
void process(UniquePtr<Widget> w);

Widget* raw = new Widget(1);
process(raw);   // WITHOUT explicit: silently wraps raw in UniquePtr
                // UniquePtr takes ownership, destroys Widget when process() ends
                // raw is now dangling!
```

With `explicit`, the compiler forces you to be intentional:

```cpp
process(raw);                       // ✗ compile error
process(UniquePtr<Widget>(raw));    // ✓ explicit — you clearly mean to transfer ownership
```

This prevents accidental ownership transfers. The same reason `shared_ptr` and `string_view` constructors from raw pointers are explicit (or restricted).

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 Q2: Move assignment — why do we need to delete the old resource?</h2>

Consider:

```cpp
UniquePtr<Widget> a(new Widget(3));   // a owns Widget(3)
UniquePtr<Widget> b(new Widget(4));   // b owns Widget(4)

a = std::move(b);   // a should now own Widget(4)
                     // but what happens to Widget(3)?
```

If your move assignment just steals `b`'s pointer without deleting `a`'s current one:

```cpp
// WRONG:
UniquePtr& operator=(UniquePtr&& other) noexcept {
    ptr_ = other.ptr_;           // Widget(3) is leaked!
    other.ptr_ = nullptr;
    return *this;
}
```

Correct implementation:

```cpp
UniquePtr& operator=(UniquePtr&& other) noexcept {
    if (this != &other) {
        reset(other.release());   // delete old, steal new
    }
    return *this;
}
```

Or equivalently:

```cpp
UniquePtr& operator=(UniquePtr&& other) noexcept {
    if (this != &other) {
        if (ptr_) deleter_(ptr_);  // clean up old resource
        ptr_ = other.ptr_;         // steal
        other.ptr_ = nullptr;      // null out source
    }
    return *this;
}
```

This is different from your Day 1 `String` move assignment where the copy-and-swap idiom handled cleanup automatically (the old data ended up in `other` and was destroyed). Here, since `unique_ptr` is **move-only** (no copy), you can't use copy-and-swap — there's no copy constructor to create the by-value parameter.

<br>

#### Wait — can't we use swap for move assignment instead?

Actually, you can:

```cpp
UniquePtr& operator=(UniquePtr&& other) noexcept {
    UniquePtr temp(std::move(other));  // move-construct temp from other
    swap(*this, temp);                  // swap: *this gets other's ptr, temp gets old ptr
    return *this;                       // temp destroyed → deletes old resource
}
```

This works and is exception-safe. But it's slightly less efficient — you're doing a swap (3 pointer operations) when a simple steal + null would suffice. For `unique_ptr`, the direct approach is cleaner.

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 Q3: <code>release()</code> vs <code>reset()</code> — when to use which?</h2>

These are frequently confused:

```
release():  "I'm giving up ownership. Someone else must delete this."
            Returns the raw pointer. UniquePtr becomes nullptr.
            The resource is NOT deleted.

reset():    "Delete what I currently own. Optionally take something new."
            The OLD resource IS deleted.
            Optionally takes a new raw pointer to own.
```

<br>

#### When to use `release()`

- Passing ownership to a C API that takes raw pointers and will `free()` later
- Transferring to a different smart pointer type
- Moving ownership into a container that stores raw pointers

```cpp
auto widget = std::make_unique<Widget>(42);
Widget* raw = widget.release();    // widget is nullptr now
legacy_api_takes_ownership(raw);   // C function that will free it later
```

<br>

#### When to use `reset()`

- Explicitly destroying the resource early (before scope end)
- Replacing the managed object with a different one

```cpp
auto conn = std::make_unique<Connection>("db.host");
conn.reset();                         // close connection NOW, don't wait for scope end
conn.reset(new Connection("other"));  // open a new connection
```

<br>

#### Common mistake

```cpp
auto p = std::make_unique<int>(42);
p.release();    // BUG: returned pointer is ignored → MEMORY LEAK
                // release() does NOT delete! It just lets go.
```

If you want to delete early, use `reset()`, not `release()`.

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 Q4: Why can't <code>unique_ptr</code> use copy-and-swap like our String?</h2>

In Day 1, our `String` had:

```cpp
String& operator=(String other) {   // by VALUE — copies or moves
    swap(*this, other);
    return *this;
}
```

This works because `String` has **both** a copy constructor and a move constructor. When passed by value, the compiler picks the right one.

But `unique_ptr`'s copy constructor is **deleted**:

```cpp
UniquePtr(const UniquePtr&) = delete;
```

So if you wrote:

```cpp
UniquePtr& operator=(UniquePtr other) {   // by VALUE
    swap(*this, other);
    return *this;
}
```

The only way to create `other` would be via move constructor. This would actually work for move assignment:

```cpp
UniquePtr<Widget> a(new Widget(1));
UniquePtr<Widget> b(new Widget(2));
a = std::move(b);   // 'other' move-constructed from b → swap → temp destroyed
```

But it would **fail for copy assignment** (which is what we want — copies should be forbidden). So technically this would compile and work correctly. The real-world `std::unique_ptr` uses a separate `operator=(UniquePtr&&)` because:

1. It makes the move-only intent explicit in the interface
2. It avoids the unnecessary temp + swap overhead
3. It's clearer to readers of the code

<br><br>

---
---

# PART 5: REFLECT & NOTES (10 min)

---

<br>

Write answers in your own notes:

1. **Why is `unique_ptr` zero-cost compared to a raw pointer?**
   - With a stateless deleter (like `DefaultDeleter`), EBO ensures no extra storage. The destructor call to `delete` is the same code you'd write manually.

2. **When would you prefer `shared_ptr` over `unique_ptr`?**
   - When multiple owners need to share the resource and you can't determine a single clear owner. But prefer `unique_ptr` by default — it's simpler, faster, and enforces clearer ownership.

3. **What's the danger of `get()`?**
   - The returned raw pointer is non-owning. If the `unique_ptr` is moved, reset, or destroyed, the raw pointer dangles. Never store the result of `get()` long-term.

4. **Can you put `unique_ptr` in a `std::vector`?**
   - Yes! `vector::push_back(std::move(ptr))` works because `unique_ptr` is movable. `vector::push_back(ptr)` (without move) won't compile.

5. **Why use `make_unique` instead of `new`?**
   - Exception safety (pre-C++17), no raw `new` in application code, shorter syntax with `auto`.

<br><br>

---
---

# INTERVIEW QUESTIONS TO PRACTICE

---

<br>

1. "What is `unique_ptr`? How does it differ from `shared_ptr`?"
2. "Implement a `unique_ptr` from scratch."
3. "Why is `unique_ptr`'s copy constructor deleted?"
4. "How would you implement a factory function that returns a polymorphic object?"
5. "What is a custom deleter? Give a real-world example."
6. "Is `unique_ptr` zero-cost? Prove it."
7. "What happens if you call `release()` and don't delete the pointer?"

<br><br>

---
---

# REFERENCE SOLUTION

---

<br>

<details>
<summary>Click to expand the complete UniquePtr implementation</summary>

<br>

```cpp
#include <iostream>
#include <utility>
#include <cstdio>
#include <type_traits>

template <typename T>
struct DefaultDeleter {
    void operator()(T* ptr) const {
        std::cout << "  [DefaultDeleter]\n";
        delete ptr;
    }
};

template <typename T, typename Deleter = DefaultDeleter<T>>
class UniquePtr : private Deleter {
private:
    T* ptr_;

public:
    explicit UniquePtr(T* ptr = nullptr, Deleter d = Deleter())
        : Deleter(d), ptr_(ptr)
    {
        std::cout << "  [ctor] UniquePtr\n";
    }

    ~UniquePtr() {
        if (ptr_) {
            std::cout << "  [dtor] UniquePtr → calling deleter\n";
            Deleter::operator()(ptr_);
        }
    }

    UniquePtr(const UniquePtr&) = delete;
    UniquePtr& operator=(const UniquePtr&) = delete;

    UniquePtr(UniquePtr&& other) noexcept
        : Deleter(std::move(other))
        , ptr_(other.ptr_)
    {
        std::cout << "  [move ctor] UniquePtr\n";
        other.ptr_ = nullptr;
    }

    UniquePtr& operator=(UniquePtr&& other) noexcept {
        std::cout << "  [move assign] UniquePtr\n";
        if (this != &other) {
            reset();
            ptr_ = other.ptr_;
            other.ptr_ = nullptr;
        }
        return *this;
    }

    T* get() const noexcept { return ptr_; }

    T& operator*() const { return *ptr_; }
    T* operator->() const noexcept { return ptr_; }

    explicit operator bool() const noexcept { return ptr_ != nullptr; }

    T* release() noexcept {
        T* tmp = ptr_;
        ptr_ = nullptr;
        return tmp;
    }

    void reset(T* ptr = nullptr) noexcept {
        T* old = ptr_;
        ptr_ = ptr;
        if (old) Deleter::operator()(old);
    }

    friend void swap(UniquePtr& a, UniquePtr& b) noexcept {
        using std::swap;
        swap(a.ptr_, b.ptr_);
    }
};

template <typename T, typename... Args>
UniquePtr<T> make_unique(Args&&... args) {
    return UniquePtr<T>(new T(std::forward<Args>(args)...));
}

// --- Test ---
struct Widget {
    int id;
    Widget(int i) : id(i) { std::cout << "  Widget(" << id << ") constructed\n"; }
    ~Widget() { std::cout << "  Widget(" << id << ") destroyed\n"; }
    void greet() { std::cout << "  Hello from Widget " << id << "\n"; }
};

int main() {
    std::cout << "=== 1. Basic ===\n";
    { UniquePtr<Widget> p(new Widget(1)); p->greet(); }

    std::cout << "\n=== 2. Move ctor ===\n";
    {
        UniquePtr<Widget> a(new Widget(2));
        UniquePtr<Widget> b = std::move(a);
        std::cout << "  a null: " << (a.get() == nullptr) << "\n";
        b->greet();
    }

    std::cout << "\n=== 3. Move assign ===\n";
    {
        UniquePtr<Widget> a(new Widget(3));
        UniquePtr<Widget> b(new Widget(4));
        a = std::move(b);
        a->greet();
    }

    std::cout << "\n=== 4. release ===\n";
    {
        UniquePtr<Widget> p(new Widget(5));
        Widget* raw = p.release();
        std::cout << "  p null: " << (p.get() == nullptr) << "\n";
        delete raw;
    }

    std::cout << "\n=== 5. reset ===\n";
    {
        UniquePtr<Widget> p(new Widget(6));
        p.reset(new Widget(7));
        p.reset();
    }

    std::cout << "\n=== 6. Custom deleter ===\n";
    {
        struct FileCloser {
            void operator()(FILE* f) const {
                std::cout << "  [FileCloser]\n";
                if (f) fclose(f);
            }
        };
        UniquePtr<FILE, FileCloser> f(fopen("/dev/null", "r"));
        std::cout << "  valid: " << (f ? "yes" : "no") << "\n";
    }

    std::cout << "\n=== 7. make_unique ===\n";
    { auto p = make_unique<Widget>(99); p->greet(); }

    std::cout << "\n=== 8. Zero-cost check ===\n";
    std::cout << "  sizeof(UniquePtr<int>)  = " << sizeof(UniquePtr<int>) << "\n";
    std::cout << "  sizeof(int*)            = " << sizeof(int*) << "\n";
    static_assert(sizeof(UniquePtr<int>) == sizeof(int*), "zero-cost abstraction");

    std::cout << "\n=== Done ===\n";
    return 0;
}
```

</details>

<br>

---

**Next**: [Day 3 — `shared_ptr` Deep Dive →](day03-shared-ptr.md)  *(coming soon)*
