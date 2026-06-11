# Day 51: Type Erasure

[← Back to Study Plan](../lld-study-plan.md) | [← Day 50](day50-crtp.md)

> **Time**: ~1.5-2 hours
> **Goal**: Type erasure is the technique of hiding a concrete type behind a uniform interface *without* forcing that type to inherit from a common base — you store *any* type that satisfies a duck-typed contract and dispatch to it through a hand-built vtable. It is how `std::function`, `std::any`, and `std::shared_ptr`'s deleter work. Learn the **concept/model idiom** (an internal abstract `Concept` plus a templated `Model<T>`), why it gives "value semantics over an interface," the **small-buffer optimization (SBO)** that avoids heap allocation for small callables, and how to build a hand-rolled vtable. Build a `Function<R(Args...)>` from scratch, with SBO.

---
---

# PART 1: WHAT PROBLEM DOES TYPE ERASURE SOLVE?

---
---

<br>

<h2 style="color: #2980B9;">📘 51.1 The Core Idea</h2>

You want to store and call "anything that looks like a function `int(int)`": a free function, a lambda, a functor object, a `std::bind` result. They share **no common base class**, have **different sizes**, and are **unrelated types**. Yet you want one type — `Function<int(int)>` — that can hold any of them and be passed around by value.

```cpp
Function<int(int)> f;
f = [](int x){ return x + 1; };       // a lambda
f = &some_free_function;              // a function pointer
f = MyFunctor{};                      // a class with operator()
int y = f(41);                        // call uniformly
```

Type erasure "erases" the concrete type at the boundary: the *caller* sees only `Function<int(int)>`; the *concrete type* is captured internally and reached through a generated dispatch mechanism.

```
                       ┌──────────────────────────────────┐
   user's lambda  ───► │  Function<int(int)>              │
   (concrete T)        │   ├─ a hand-built vtable (call,   │
                       │   │   clone, destroy, ...)        │
                       │   └─ storage for the T (SBO or    │
                       │       heap)                       │
                       └──────────────────────────────────┘
                                     │ f(41)
                                     ▼
                          vtable.call(storage, 41)  → T::operator()(41)
```

<br>

#### Type erasure vs the alternatives

| Approach | Common base required? | Value semantics? | Heterogeneous? | Cost |
|----------|----------------------|------------------|----------------|------|
| **Virtual inheritance** | Yes — every type must derive from `IFoo` | No (pointer/reference semantics) | Yes | vtable |
| **CRTP** (Day 50) | No, but each type is a *distinct* type | Yes | **No** (can't mix) | none |
| **`std::variant`** | No, but the type set is *closed* (fixed) | Yes | Closed set only | tagged union |
| **Type erasure** | **No** — duck-typed, open set | **Yes** | **Yes** | one indirection |

Type erasure is the unique combination of "no inheritance required," "value semantics," "open/heterogeneous set." That's why the standard library uses it for `std::function`, `std::any`, and friends.

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 51.2 Value Semantics Over a Polymorphic Interface</h2>

The killer feature: a type-erased object behaves like a *value* — copyable, movable, assignable, storable in containers by value — even though it dispatches polymorphically. Sean Parent's famous phrase is **"polymorphism is a property of code, not of types."** Instead of forcing `Circle`/`Square` to inherit from `Shape`, you write them as plain value types and *erase* them at the point of use:

```cpp
std::vector<Drawable> shapes;     // Drawable is type-erased
shapes.push_back(Circle{2.0});    // Circle never heard of Drawable
shapes.push_back(Square{3.0});
for (auto& s : shapes) s.draw();  // dispatches to the right draw()
```

`Circle` and `Square` are non-intrusive — no base class, no virtual, no awareness of `Drawable`. This decouples the *concept* from the *types* that model it.

<br><br>

---
---

# PART 2: HOW IT WORKS — THE CONCEPT/MODEL IDIOM

---
---

<br>

<h2 style="color: #2980B9;">📘 51.3 Anatomy</h2>

The canonical implementation has three pieces:

```cpp
class Drawable {
    // 1) CONCEPT: the internal interface (abstract base) describing
    //    what every held object must be able to do.
    struct Concept {
        virtual ~Concept() = default;
        virtual void draw() const = 0;
        virtual std::unique_ptr<Concept> clone() const = 0;   // for value copy
    };

    // 2) MODEL<T>: a templated adapter that holds a concrete T and
    //    forwards the interface to T's own methods (duck typing).
    template <typename T>
    struct Model final : Concept {
        T obj;
        explicit Model(T x) : obj(std::move(x)) {}
        void draw() const override { obj.draw(); }
        std::unique_ptr<Concept> clone() const override {
            return std::make_unique<Model>(obj);
        }
    };

    std::unique_ptr<Concept> self_;   // 3) the erased storage

public:
    // Templated constructor: accepts ANY T that has draw()
    template <typename T>
    Drawable(T x) : self_(std::make_unique<Model<T>>(std::move(x))) {}

    // Value semantics: deep copy through clone()
    Drawable(const Drawable& o) : self_(o.self_->clone()) {}
    Drawable(Drawable&&) noexcept = default;
    Drawable& operator=(Drawable o) { self_ = std::move(o.self_); return *this; }

    void draw() const { self_->draw(); }
};
```

- **Concept** is the *erased* interface — invisible to users. It is a normal abstract class with virtuals.
- **Model<T>** is the *bridge*: it inherits the Concept and forwards each virtual to `T`'s same-named method. One `Model` instantiation per concrete `T`.
- The outer class owns a `unique_ptr<Concept>` and exposes a *value-typed*, *inheritance-free* public API.

The templated constructor is where erasure happens: the moment you pass a `Circle`, the compiler stamps out `Model<Circle>`, captures the `Circle`, and stores it as a `Concept*`. From then on the `Circle` type is gone from the type system; only the vtable remembers how to draw it.

<br>

#### Why `clone()`?

`unique_ptr<Concept>` is move-only, but we want `Drawable` to be *copyable* (value semantics). Copying must duplicate the held `T`, but the outer class doesn't know `T`'s type anymore — so it asks the vtable: `clone()` is virtual, and `Model<T>::clone()` knows the real `T` and makes a fresh `Model<T>`.

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 51.4 The Manual Vtable View</h2>

The `Concept`/`Model` design *uses* the compiler's vtable. But you can build the vtable **by hand** — a struct of function pointers — which is exactly how high-performance `std::function` implementations avoid the abstract-base machinery and how you'd do it in C. Understanding this demystifies "what a vtable actually is."

```cpp
struct VTable {
    int  (*call)   (void* obj, int arg);   // invoke
    void (*destroy)(void* obj);            // run T's destructor
    void*(*clone)  (void* obj);            // copy-construct a new T
};

template <typename T>
const VTable* vtable_for() {
    static const VTable vt = {
        [](void* o, int a) { return (*static_cast<T*>(o))(a); },
        [](void* o)        { delete static_cast<T*>(o); },
        [](void* o)        { return static_cast<void*>(new T(*static_cast<T*>(o))); }
    };
    return &vt;                            // one shared vtable per T — like the compiler's
}
```

`vtable_for<T>()` returns a pointer to a `static` table — exactly one per `T`, shared by all instances, just like a compiler-generated vtable. The captureless lambdas decay to function pointers. This is the engine room of type erasure with **no inheritance at all**.

<br><br>

---
---

# PART 3: SMALL-BUFFER OPTIMIZATION (SBO)

---
---

<br>

<h2 style="color: #2980B9;">📘 51.5 The Allocation Problem</h2>

The naïve `Concept`/`Model` design heap-allocates *every* held object via `make_unique`. But most callables are tiny — a lambda capturing one pointer is 8 bytes. Heap-allocating 8 bytes on every `std::function` assignment in a hot loop is wasteful.

**SBO**: reserve a fixed inline buffer inside the type-erased object. If `T` fits (and is suitably trivially relocatable / nothrow-movable), construct it *in place* in the buffer — no heap. If it's too big, fall back to heap.

```
┌─── Function<int(int)> ───────────────────────────┐
│  const VTable* vtbl;                              │
│  union {                                          │
│     alignas(MAX) char buf[SBO_SIZE];  ← small T   │
│     void*            heap;            ← large T   │
│  };                                               │
│  bool on_heap;                                    │
└───────────────────────────────────────────────────┘
```

Real `std::function` implementations (libstdc++, libc++, MSVC) all do this; the inline buffer is typically large enough for a small lambda or a member-function-pointer + object pointer (≈ 16–32 bytes).

<br>

#### The SBO decision

```cpp
static constexpr std::size_t SBO_SIZE = 32;

template <typename T>
static constexpr bool fits_sbo =
       sizeof(T)  <= SBO_SIZE
    && alignof(T) <= alignof(std::max_align_t)
    && std::is_nothrow_move_constructible_v<T>;   // must relocate without throwing
```

The `is_nothrow_move_constructible` requirement matters: when the erased object itself moves, we move the inline `T` from old buffer to new buffer, and that move must not throw (no exception-safe rollback is possible mid-relocation).

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 51.6 SBO Trade-offs</h2>

| Pro | Con |
|-----|-----|
| No heap allocation for small callables (the common case) | Larger sizeof for the erased object (carries the buffer even when empty/heap) |
| Better cache locality — callable lives inline | More complex move/copy/destroy logic (two paths) |
| Faster construct/destroy in hot loops | Buffer size is a tuning compromise; too big wastes memory |

Picking `SBO_SIZE`: large enough for the typical capture (a `this` pointer plus one or two captured values), small enough not to bloat every stored `Function`. 16–32 bytes is the usual sweet spot.

<br><br>

---
---

# PART 4: BUILD EXERCISE

---
---

<br>

<h2 style="color: #2980B9;">📘 51.7 Exercise: Build `Function<R(Args...)>` With SBO</h2>

**Motivation.** Re-implement the core of `std::function`: a value-semantic, copyable callable wrapper that holds any compatible callable, dispatches through a hand-built vtable, and uses SBO to skip the heap for small callables. This single exercise touches every concept on this page — the concept/model idiom, a manual vtable, value semantics via `clone`, and SBO.

We use a hand-built vtable (function-pointer struct) rather than an abstract base, both because it's faster and because it makes the mechanics explicit.

<br>

#### Skeleton

```cpp
// function.h
#pragma once
#include <cstddef>
#include <new>
#include <type_traits>
#include <utility>
#include <stdexcept>

template <typename Signature>
class Function;                                  // primary template (undefined)

template <typename R, typename... Args>
class Function<R(Args...)> {
    static constexpr std::size_t SBO_SIZE = 32;

    // Hand-built vtable: one shared instance per concrete callable type.
    struct VTable {
        R    (*call)   (void* self, Args&&... args);
        void (*destroy)(void* self);
        void (*copy)   (const void* src, void* dstStorage, bool dstHeap);
        void (*move)   (void* src,       void* dstStorage, bool dstHeap);
    };

    template <typename T>
    static constexpr bool fits_sbo =
           sizeof(T)  <= SBO_SIZE
        && alignof(T) <= alignof(std::max_align_t)
        && std::is_nothrow_move_constructible_v<T>;

    // Build the static vtable for a concrete callable T.
    template <typename T>
    static const VTable* vtable_for() {
        static const VTable vt = {
            // call
            [](void* self, Args&&... args) -> R {
                return (*static_cast<T*>(self))(std::forward<Args>(args)...);
            },
            // destroy
            [](void* self) { static_cast<T*>(self)->~T(); },
            // copy: construct a copy of *src into dst's storage
            [](const void* src, void* dstStorage, bool /*dstHeap*/) {
                ::new (dstStorage) T(*static_cast<const T*>(src));
            },
            // move: relocate *src into dst's storage
            [](void* src, void* dstStorage, bool /*dstHeap*/) {
                ::new (dstStorage) T(std::move(*static_cast<T*>(src)));
            }
        };
        return &vt;
    }

    union {
        alignas(std::max_align_t) char buf_[SBO_SIZE];   // inline storage
        void* heap_;                                     // heap pointer
    };
    const VTable* vtbl_ = nullptr;   // nullptr == empty
    bool on_heap_ = false;

    // Returns the address where the T currently lives.
    void*       storage()       { return on_heap_ ? heap_ : static_cast<void*>(buf_); }
    const void* storage() const { return on_heap_ ? heap_ : static_cast<const void*>(buf_); }

    // Place a copy/move of a T (already constructed elsewhere) into THIS object.
    // Used by copy/move ctors. allocate() decides SBO vs heap based on T.
    template <typename T>
    void* allocate() {
        if constexpr (fits_sbo<T>) { on_heap_ = false; return buf_; }
        else { heap_ = ::operator new(sizeof(T)); on_heap_ = true; return heap_; }
    }

public:
    Function() noexcept = default;
    Function(std::nullptr_t) noexcept {}

    // Construct from any compatible callable T.
    template <typename T,
              typename DT = std::decay_t<T>,
              typename = std::enable_if_t<!std::is_same_v<DT, Function>>>
    Function(T&& t) {
        void* dst = allocate<DT>();
        ::new (dst) DT(std::forward<T>(t));
        vtbl_ = vtable_for<DT>();
    }

    Function(const Function& o) {
        if (o.vtbl_) {
            // Allocate same-size storage, then ask the vtable to copy.
            void* dst = o.on_heap_ ? (heap_ = ::operator new(/*size?*/ SBO_SIZE), heap_)
                                   : static_cast<void*>(buf_);
            on_heap_ = o.on_heap_;
            o.vtbl_->copy(o.storage(), dst, on_heap_);
            vtbl_ = o.vtbl_;
        }
    }

    Function(Function&& o) noexcept {
        if (o.vtbl_) {
            if (o.on_heap_) {                       // steal the pointer
                heap_ = o.heap_; on_heap_ = true;
            } else {                                // relocate inline storage
                o.vtbl_->move(o.storage(), buf_, false);
                o.vtbl_->destroy(o.storage());
                on_heap_ = false;
            }
            vtbl_ = o.vtbl_;
            o.vtbl_ = nullptr;
        }
    }

    Function& operator=(Function o) noexcept {      // copy-and-swap
        swap(o);
        return *this;
    }

    ~Function() { reset(); }

    void reset() noexcept {
        if (vtbl_) {
            vtbl_->destroy(storage());
            if (on_heap_) ::operator delete(heap_);
            vtbl_ = nullptr;
            on_heap_ = false;
        }
    }

    void swap(Function& o) noexcept {
        // Simplest correct swap: move through a temporary.
        Function tmp(std::move(*this));
        *this = nullptr; this->moveFrom(std::move(o));
        o = nullptr;     o.moveFrom(std::move(tmp));
    }

    R operator()(Args... args) const {
        if (!vtbl_) throw std::bad_function_call();
        return vtbl_->call(const_cast<void*>(storage()),
                           std::forward<Args>(args)...);
    }

    explicit operator bool() const noexcept { return vtbl_ != nullptr; }

private:
    void moveFrom(Function&& o) noexcept { *this = Function(std::move(o)); }
};
```

> The skeleton above is intentionally *almost* complete with a couple of rough edges (the heap copy needs the real `T` size; `swap` is sketched) so you have something to finish. Your job in the exercise is to make it compile cleanly and pass the tests — store the heap allocation size, or store a `size`/`align` in the vtable so the copy ctor allocates correctly. A clean approach: add `std::size_t size; std::size_t align;` fields to `VTable` and use them in the copy constructor.

<br>

#### Test driver

```cpp
// main.cpp
#include "function.h"
#include <iostream>
#include <string>
#include <cassert>

int free_add_one(int x) { return x + 1; }

struct Multiplier {
    int factor;
    int operator()(int x) const { return x * factor; }
};

int main() {
    std::cout << "=== sizeof ===\n";
    std::cout << "  sizeof(Function<int(int)>) = "
              << sizeof(Function<int(int)>) << " (carries SBO buffer)\n";

    std::cout << "\n=== free function ===\n";
    Function<int(int)> f = &free_add_one;
    std::cout << "  f(41) = " << f(41) << "\n";        // 42
    assert(f(41) == 42);

    std::cout << "\n=== lambda (fits SBO) ===\n";
    int base = 100;
    f = [base](int x){ return x + base; };
    std::cout << "  f(5) = " << f(5) << "\n";          // 105
    assert(f(5) == 105);

    std::cout << "\n=== functor ===\n";
    f = Multiplier{3};
    std::cout << "  f(7) = " << f(7) << "\n";          // 21
    assert(f(7) == 21);

    std::cout << "\n=== value semantics: copy ===\n";
    Function<int(int)> g = f;                          // deep copy
    f = nullptr;                                       // reset original
    std::cout << "  g(7) = " << g(7) << " (copy survives original reset)\n";
    assert(g(7) == 21);

    std::cout << "\n=== move ===\n";
    Function<int(int)> h = std::move(g);
    std::cout << "  h(7) = " << h(7) << "\n";
    std::cout << "  g empty? " << std::boolalpha << !static_cast<bool>(g) << "\n";

    std::cout << "\n=== empty call throws ===\n";
    Function<int(int)> empty;
    try { empty(1); }
    catch (const std::exception& e) { std::cout << "  caught: bad_function_call\n"; }

    std::cout << "\n=== large capture (forces heap) ===\n";
    std::string big(200, 'x');                         // > SBO_SIZE
    Function<std::size_t()> sizer = [big]{ return big.size(); };
    std::cout << "  sizer() = " << sizer() << "\n";    // 200
    assert(sizer() == 200);

    std::cout << "\nAll assertions passed.\n";
    return 0;
}
```

<br>

#### Compile and run

```bash
g++ -std=c++17 -Wall -Wextra -Wpedantic -fsanitize=address \
    -o day51 main.cpp && ./day51
```

<br>

#### Expected output pattern

```
=== sizeof ===
  sizeof(Function<int(int)>) = 48 (carries SBO buffer)
=== free function ===
  f(41) = 42
=== lambda (fits SBO) ===
  f(5) = 105
=== functor ===
  f(7) = 21
=== value semantics: copy ===
  g(7) = 21 (copy survives original reset)
=== move ===
  h(7) = 21
  g empty? true
=== empty call throws ===
  caught: bad_function_call
=== large capture (forces heap) ===
  sizer() = 200
All assertions passed.
```

<br>

#### Bonus Challenges

1. **Fix the copy ctor properly.** Add `size`/`align` (or a single `clone` that does its own allocation) so the heap path copies the right number of bytes. Run under ASan to prove no overflow.

2. **`target<T>()` accessor.** Add `template<class T> T* target()` returning a pointer to the held object if the stored type is `T`, else `nullptr`. You'll need to compare vtable pointers (store a type tag) — a mini `std::any`.

3. **Move-only callables.** `std::function` *requires* copyability. Build a `MoveOnlyFunction` variant that drops `copy` from the vtable, allowing capture of `unique_ptr`. (This is C++23's `std::move_only_function`.)

4. **Measure SBO wins.** Benchmark assigning a small lambda 10M times with your SBO version vs a version that always heap-allocates. Report allocations (use a counting `operator new`).

5. **`Concept`/`Model` rewrite.** Re-implement the same `Function` using an abstract `Concept` base + `Model<T>` instead of a hand-built vtable. Compare clarity and codegen.

6. **`std::any` from scratch.** Using the same erasure machinery (vtable with destroy/copy/type-id), build a minimal `Any` that stores any copyable type and supports `any_cast<T>`.

<br><br>

---
---

# PART 5: DEEP DIVE — COMMON QUESTIONS

---
---

<br>

<h2 style="color: #2980B9;">📘 51.8 Q&A</h2>

<br>

#### Q1: "How does `std::function` actually work?"

It is type erasure with SBO. Internally it keeps a small inline buffer and either a hand-built vtable (function pointers for invoke/copy/move/destroy) or a `Concept`/`Model` hierarchy, depending on the implementation. Small callables live in the buffer; large ones spill to the heap. Calling goes through one indirect call.

<br>

#### Q2: "What's the difference between type erasure and a `std::variant`?"

`variant` is a *closed* set — you must enumerate every alternative type at compile time, and it's a tagged union (no heap, no vtable, dispatch by `visit`). Type erasure is an *open* set — any type satisfying the duck-typed contract works, even ones defined later — at the cost of an indirection (and possibly an allocation).

<br>

#### Q3: "Why does the outer class need `clone()` / a copy in the vtable?"

To preserve **value semantics**. The outer object owns the erased thing but doesn't know its static type, so it cannot itself copy-construct it. The vtable carries a `copy`/`clone` entry that *does* know `T` and performs the deep copy. Without it, the type-erased wrapper would be move-only.

<br>

#### Q4: "What exactly is 'erased' — when does the type information disappear?"

At the templated constructor. The constructor is instantiated per `T`, captures the `T`, and stamps out the `Model<T>` / `vtable_for<T>()`. After that, the only handle to `T` is the function pointers in the vtable; the static type is gone from the public interface. The information isn't *lost* — it's *frozen into code* (the vtable's lambdas).

<br>

#### Q5: "Why is `is_nothrow_move_constructible` required for SBO?"

When the erased object moves, the inline `T` must be relocated from the old buffer to the new buffer. If that move could throw, you'd be mid-relocation with no safe rollback (the old buffer's bytes may already be clobbered). Standard library SBO falls back to the heap for throwing-move types, where moving the *pointer* is always noexcept.

<br>

#### Q6: "Is type erasure slower than virtual inheritance?"

Roughly the same dispatch cost — one indirection. Type erasure may add an allocation (mitigated by SBO) and a copy (`clone`), but it buys non-intrusiveness (types need no base class) and value semantics. If you control the types and they can share a base, plain virtual is simpler; if you can't (third-party types, or you want value semantics), erasure wins.

<br>

#### Q7: "Can I do type erasure with no heap and no vtable at all?"

For a *closed* set, yes — that's `std::variant`. For an *open* set you need *some* indirection: either an allocation, or SBO (still needs a vtable for dispatch). You can't have open-set polymorphism with zero indirection — that's fundamentally what CRTP can't do either (Day 50).

<br>

#### Q8: "How is this related to Sean Parent's 'inheritance is the base class of evil' talk?"

That talk popularized using type erasure to get polymorphism *without* intrusive inheritance: write your types as plain values, and erase them behind a `Drawable`/`Document` wrapper at the use site. Polymorphism becomes a property of the *interface code*, not baked into the *types*. Our `Drawable` (§51.3) is exactly that pattern.

<br><br>

---

## Reflection Questions

1. Name the three parts of the concept/model idiom and what each is responsible for.
2. Why does a type-erased wrapper need a `clone`/`copy` entry to be copyable, and where does that copy actually happen?
3. What does "value semantics over a polymorphic interface" mean, and why is it desirable?
4. Explain the SBO decision: which properties of `T` make it eligible for the inline buffer, and why is nothrow-move required?
5. How is a hand-built vtable (struct of function pointers) equivalent to the compiler's vtable? Where does the "one table per type" come from?
6. When would you choose `std::variant` over type erasure, and vice versa?

---

## Interview Questions

1. "Explain how `std::function` is implemented internally."
2. "Implement a minimal `Function<R(Args...)>` from scratch."
3. "What is the concept/model idiom? Sketch the three classes."
4. "What is the small-buffer optimization and what determines whether a callable uses it?"
5. "Build a type-erased `Drawable` that stores `Circle`/`Square` without a common base."
6. "Compare type erasure, virtual inheritance, CRTP, and `std::variant`."
7. "Why must SBO require nothrow-move-constructible types?"
8. "How would you make a *move-only* type-erased callable (capture a `unique_ptr`)?"
9. "Build a minimal `std::any` and `any_cast`."
10. "Where exactly is the concrete type 'erased,' and is the information truly lost?"

---

**Next**: Day 52 — Policy-Based Design & Concepts →
