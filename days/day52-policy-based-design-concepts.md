# Day 52: Policy-Based Design & Concepts

[← Back to Study Plan](../lld-study-plan.md) | [← Day 51](day51-type-erasure.md)

> **Time**: ~1.5-2 hours
> **Goal**: Policy-based design (Andrei Alexandrescu, *Modern C++ Design*) is the technique of composing a class's behavior from orthogonal **policy** template parameters — each policy is a small class that implements one design decision (ownership, threading, checking, storage), and the **host class** inherits or contains them to assemble the final type at compile time. It's "compile-time strategy": you pick algorithms by *type*, with zero runtime cost. Then learn how C++20 **concepts** turn unchecked "duck-typed" template parameters into *enforced contracts* with readable errors, using `requires` clauses to constrain each policy. Build a policy-based smart pointer whose ownership and threading behavior are pluggable, with every policy constrained by a concept. Compile with `-std=c++20`.

---
---

# PART 1: WHAT IS POLICY-BASED DESIGN?

---
---

<br>

<h2 style="color: #2980B9;">📘 52.1 The Core Idea</h2>

A real class usually bundles several *independent* decisions:

- A smart pointer must decide: **ownership** (unique? shared? intrusive?), **threading** (atomic refcount or not?), **null-checking** (assert on deref or not?).
- A container must decide: **allocation** strategy, **growth** policy, **bounds checking**.

Policy-based design makes each decision a **template parameter** — a policy class — and lets the **host class** combine them. Instead of one monolithic class or an explosion of `bool` flags, you get a single configurable template:

```cpp
template <typename T,
          typename OwnershipPolicy,
          typename ThreadingPolicy,
          typename CheckingPolicy>
class SmartPtr
    : OwnershipPolicy, ThreadingPolicy, CheckingPolicy {
    /* host glues the policies together */
};
```

Each policy is an *orthogonal axis*. Combining 3 ownership × 2 threading × 2 checking policies yields 12 distinct types — without writing 12 classes. This is the compile-time analogue of the **Strategy pattern** (Day 19): a policy is a strategy chosen by *type* at compile time rather than by an *object* at runtime.

```
        Strategy (runtime)                 Policy (compile time)
   ┌───────────────────────┐         ┌────────────────────────────┐
   │ ctx holds IStrategy*  │         │ Host<Policy1, Policy2, ...>  │
   │ virtual call → algo   │         │ inherits/contains policies   │
   │ swap at runtime       │         │ fixed at instantiation       │
   │ vtable cost           │         │ inlined, zero overhead       │
   └───────────────────────┘         └────────────────────────────┘
```

<br>

#### Policy vs flags

```cpp
// Flag soup — runtime branches, all behavior compiled in always:
SmartPtr p(ptr, /*shared=*/true, /*atomic=*/true, /*checked=*/false);

// Policy — behavior is the TYPE, branches resolved at compile time:
SmartPtr<Widget, Shared, MultiThreaded, Unchecked> p(ptr);
```

The policy version pays for *only* the behavior it selects: a single-threaded pointer carries no atomic operations at all, not even dead `if (atomic)` branches.

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 52.2 Host Classes & How Policies Combine</h2>

The **host** is the class template that takes the policies. Two ways to combine them:

| Mechanism | How | Pros | Cons |
|-----------|-----|------|------|
| **Inheritance** (`class Host : public P1, public P2`) | Host derives from each policy | Empty policies cost 0 bytes (EBO); host can call policy methods unqualified | Policy members leak into host's interface; ambiguities if names clash |
| **Composition** (`P1 p1; P2 p2;` members) | Host holds policy objects | Cleaner encapsulation; no name leakage | Empty policies still cost ≥1 byte each (no EBO) |

Alexandrescu's original `Loki` library uses inheritance (for EBO and to expose policy methods). Modern style often prefers composition with `[[no_unique_address]]` (C++20) to *recover* EBO for members:

```cpp
template <typename T, typename Threading>
class SmartPtr {
    [[no_unique_address]] Threading threading_;   // empty policy → 0 bytes (C++20)
    T* ptr_;
};
```

`[[no_unique_address]]` tells the compiler an empty member subobject may overlap with others, restoring the size advantage that inheritance got via EBO — but with composition's cleaner encapsulation.

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 52.3 What Makes a Good Policy</h2>

A policy is well-designed when:

- It captures **one** decision (orthogonal — changing it doesn't force changes to others).
- It has a **minimal, named interface** the host relies on (e.g., a threading policy provides `lock()`/`unlock()` or `increment(refcount)`).
- It is often **stateless** (so it's an empty class → zero cost), though stateful policies are allowed.
- It has a sensible **default** so users opt out only when needed.

The host should *only* talk to a policy through its documented interface — that interface is exactly what a C++20 *concept* will formalize in Part 2.

<br><br>

---
---

# PART 2: C++20 CONCEPTS — CONSTRAINING POLICIES

---
---

<br>

<h2 style="color: #2980B9;">📘 52.4 The Problem Concepts Solve</h2>

Before C++20, a policy was just a template parameter — an *unchecked* contract. Pass a type missing a required method and you get a wall of errors from *deep inside* the host, pointing at line 400 of a header, not at your mistake.

```cpp
template <typename ThreadingPolicy>
class SmartPtr {
    void retain() { ThreadingPolicy::increment(count_); }  // requires increment()
};
SmartPtr<Widget, BrokenPolicy> p;   // error blooms 5 frames deep if increment() is missing
```

A **concept** is a named, compile-time predicate over types. It lets you state "a `ThreadingPolicy` must provide `increment` and `decrement`" *up front*, so a violation is reported **at the call site** with a message naming the unmet requirement.

<br>

<h2 style="color: #2980B9;">📘 52.5 Defining and Using Concepts</h2>

```cpp
#include <concepts>

// A concept is "a bool-valued predicate on types."
template <typename P>
concept ThreadingPolicy = requires(typename P::CounterType& c) {
    { P::increment(c) } -> std::same_as<void>;   // must have static increment(CounterType&)
    { P::decrement(c) } -> std::same_as<long>;   // returns new count
    typename P::CounterType;                      // must define CounterType
};
```

The `requires` *expression* lists the operations a `P` must support. `{ expr } -> Constraint` checks that `expr` is valid *and* that its type satisfies `Constraint`. Then constrain the template:

```cpp
// Three equivalent ways to apply a concept:

// (a) as the parameter "type":
template <ThreadingPolicy P> class A {};

// (b) requires-clause after the template head:
template <typename P> requires ThreadingPolicy<P> class B {};

// (c) trailing requires:
template <typename P> class C requires ThreadingPolicy<P> {};
```

Now `SmartPtr<Widget, BrokenPolicy>` fails *immediately* with: *"constraint not satisfied: BrokenPolicy does not provide increment."* Clean, local, and self-documenting.

<br>

<h2 style="color: #2980B9;">📘 52.6 `requires` Clauses & Expressions — the Four Kinds of Requirement</h2>

Inside a `requires(...) { ... }` expression you can write four kinds of requirement:

```cpp
template <typename T>
concept Example = requires(T a, T b) {
    a + b;                              // 1. simple   — expression must be valid
    typename T::value_type;            // 2. type     — nested type must exist
    { a.size() } -> std::convertible_to<std::size_t>;  // 3. compound — valid + type constraint
    requires std::movable<T>;          // 4. nested   — another concept must hold
};
```

| Kind | Syntax | Checks |
|------|--------|--------|
| Simple | `expr;` | `expr` compiles |
| Type | `typename T::X;` | `T::X` names a type |
| Compound | `{ expr } -> Concept;` | `expr` compiles *and* its result satisfies `Concept` (and optionally `noexcept`) |
| Nested | `requires Pred<T>;` | the predicate `Pred<T>` is `true` |

These are the building blocks for expressing a policy's required interface precisely.

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 52.7 Concepts vs `static_assert` vs SFINAE</h2>

| Technique | Error quality | Overload selection | Readability |
|-----------|---------------|--------------------|-------------|
| **Nothing (raw template)** | Awful, deep in instantiation | n/a | Implicit contract |
| **`static_assert`** | Good message, but fires *after* instantiation begins | Can't drive overloading | Explicit but verbose |
| **SFINAE / `enable_if`** | Cryptic | ✅ drives overloading | Very hard to read |
| **Concepts (C++20)** | Excellent, at the call site | ✅ drives overloading, with *subsumption* (more-constrained wins) | Self-documenting |

Concepts subsume both `static_assert` (for stating requirements) and `enable_if` (for constraining overloads), with far better diagnostics. They are the modern way to constrain policies.

<br><br>

---
---

# PART 3: BUILD EXERCISE

---
---

<br>

<h2 style="color: #2980B9;">📘 52.8 Exercise: A Policy-Based Smart Pointer</h2>

**Motivation.** Build `SmartPtr<T, Ownership, Threading>` where:

- **Ownership policy** decides what happens on copy and destruction: `UniqueOwnership` (move-only, deletes on destruction) vs `RefCounted` (shared refcount, deletes when it hits zero).
- **Threading policy** decides how the refcount is mutated: `SingleThreaded` (plain `++`/`--`) vs `MultiThreaded` (`std::atomic` operations).

Both policies are constrained by C++20 concepts, so misusing them yields a crisp diagnostic. This combines policy-based design (compile-time strategy, host class, EBO via `[[no_unique_address]]`) with concepts (enforced contracts). It mirrors the real design space of `unique_ptr` vs `shared_ptr` (Day 3, `day03-shared-ptr.md`) but as *one* configurable template.

<br>

#### Skeleton

```cpp
// smart_ptr.h   — compile with -std=c++20
#pragma once
#include <atomic>
#include <concepts>
#include <cstddef>
#include <utility>

// ─────────────────────────────────────────────────────────────
// CONCEPTS that constrain the policies
// ─────────────────────────────────────────────────────────────

// A threading policy supplies a counter type and atomic-or-not inc/dec.
template <typename P>
concept ThreadingPolicy = requires {
    typename P::CounterType;
} && requires(typename P::CounterType& c) {
    { P::increment(c) } -> std::same_as<void>;
    { P::decrement(c) } -> std::same_as<long>;   // returns the NEW value
    { P::load(c)      } -> std::same_as<long>;
};

// An ownership policy decides copyability and how to release a T*.
// is_shared selects the host's copy semantics.
template <typename P, typename T>
concept OwnershipPolicy = requires(T* p) {
    { P::release(p) } -> std::same_as<void>;     // how to delete the pointee
    { P::is_shared }  -> std::convertible_to<bool>;
};

// ─────────────────────────────────────────────────────────────
// THREADING POLICIES
// ─────────────────────────────────────────────────────────────
struct SingleThreaded {
    using CounterType = long;
    static void increment(CounterType& c) { ++c; }
    static long decrement(CounterType& c) { return --c; }
    static long load(CounterType& c)      { return c; }
};

struct MultiThreaded {
    using CounterType = std::atomic<long>;
    static void increment(CounterType& c) { c.fetch_add(1, std::memory_order_relaxed); }
    static long decrement(CounterType& c) { return c.fetch_sub(1, std::memory_order_acq_rel) - 1; }
    static long load(CounterType& c)      { return c.load(std::memory_order_acquire); }
};

// ─────────────────────────────────────────────────────────────
// OWNERSHIP POLICIES
// ─────────────────────────────────────────────────────────────
template <typename T>
struct UniqueOwnership {
    static constexpr bool is_shared = false;
    static void release(T* p) { delete p; }
};

template <typename T>
struct RefCounted {
    static constexpr bool is_shared = true;
    static void release(T* p) { delete p; }
};

// ─────────────────────────────────────────────────────────────
// HOST CLASS
// ─────────────────────────────────────────────────────────────
template <typename T,
          typename Ownership = UniqueOwnership<T>,
          typename Threading = SingleThreaded>
    requires ThreadingPolicy<Threading> && OwnershipPolicy<Ownership, T>
class SmartPtr {
    T* ptr_ = nullptr;
    typename Threading::CounterType* count_ = nullptr;   // only used when shared

public:
    SmartPtr() = default;

    explicit SmartPtr(T* p) : ptr_(p) {
        if constexpr (Ownership::is_shared) {
            count_ = new typename Threading::CounterType{1};
        }
    }

    ~SmartPtr() { reset(); }

    // ---- copy: enabled only for shared ownership ----
    SmartPtr(const SmartPtr& o) {
        static_assert(Ownership::is_shared,
                      "UniqueOwnership SmartPtr is move-only; use std::move");
        ptr_ = o.ptr_; count_ = o.count_;
        if (count_) Threading::increment(*count_);
    }
    SmartPtr& operator=(const SmartPtr& o) {
        static_assert(Ownership::is_shared, "move-only");
        if (this != &o) { reset(); ptr_ = o.ptr_; count_ = o.count_;
                          if (count_) Threading::increment(*count_); }
        return *this;
    }

    // ---- move: always available ----
    SmartPtr(SmartPtr&& o) noexcept
        : ptr_(std::exchange(o.ptr_, nullptr)),
          count_(std::exchange(o.count_, nullptr)) {}
    SmartPtr& operator=(SmartPtr&& o) noexcept {
        if (this != &o) { reset();
            ptr_   = std::exchange(o.ptr_,   nullptr);
            count_ = std::exchange(o.count_, nullptr); }
        return *this;
    }

    T& operator*()  const { return *ptr_; }
    T* operator->() const { return ptr_; }
    T* get()        const { return ptr_; }
    explicit operator bool() const { return ptr_ != nullptr; }

    long use_count() const {
        if constexpr (Ownership::is_shared)
            return count_ ? Threading::load(*count_) : 0;
        else
            return ptr_ ? 1 : 0;
    }

    void reset() {
        if (!ptr_) return;
        if constexpr (Ownership::is_shared) {
            if (Threading::decrement(*count_) == 0) {
                Ownership::release(ptr_);
                delete count_;
            }
        } else {
            Ownership::release(ptr_);
        }
        ptr_ = nullptr; count_ = nullptr;
    }
};

// Convenience aliases — the "12 types from a few policies" payoff:
template <typename T> using UniquePtr      = SmartPtr<T, UniqueOwnership<T>, SingleThreaded>;
template <typename T> using SharedPtr      = SmartPtr<T, RefCounted<T>,     SingleThreaded>;
template <typename T> using AtomicSharedPtr= SmartPtr<T, RefCounted<T>,     MultiThreaded>;
```

<br>

#### Test driver

```cpp
// main.cpp   — compile with -std=c++20
#include "smart_ptr.h"
#include <iostream>
#include <cassert>

struct Widget {
    int id;
    explicit Widget(int i) : id(i) { std::cout << "  Widget " << id << " ctor\n"; }
    ~Widget()                      { std::cout << "  Widget " << id << " dtor\n"; }
    void hello() const             { std::cout << "  Widget " << id << " hello\n"; }
};

int main() {
    std::cout << "=== UniquePtr (move-only, single-threaded) ===\n";
    {
        UniquePtr<Widget> a(new Widget(1));
        a->hello();
        UniquePtr<Widget> b = std::move(a);     // move OK
        assert(!a);
        b->hello();
        // UniquePtr<Widget> c = b;             // would static_assert: move-only
    }   // Widget 1 destroyed here

    std::cout << "\n=== SharedPtr (refcounted, single-threaded) ===\n";
    {
        SharedPtr<Widget> p(new Widget(2));
        std::cout << "  use_count = " << p.use_count() << "\n";   // 1
        {
            SharedPtr<Widget> q = p;             // copy bumps refcount
            std::cout << "  use_count = " << p.use_count() << "\n"; // 2
            q->hello();
        }                                        // q dies, count -> 1
        std::cout << "  use_count = " << p.use_count() << "\n";   // 1
    }   // Widget 2 destroyed at count 0

    std::cout << "\n=== AtomicSharedPtr (refcounted, multi-threaded counter) ===\n";
    {
        AtomicSharedPtr<Widget> p(new Widget(3));
        AtomicSharedPtr<Widget> q = p;
        std::cout << "  use_count = " << p.use_count() << "\n";   // 2 (atomic)
    }   // Widget 3 destroyed

    std::cout << "\n=== sizeof: single-threaded vs atomic counter ===\n";
    std::cout << "  sizeof(SharedPtr<Widget>)       = " << sizeof(SharedPtr<Widget>)       << "\n";
    std::cout << "  sizeof(AtomicSharedPtr<Widget>) = " << sizeof(AtomicSharedPtr<Widget>) << "\n";

    std::cout << "\nAll done.\n";
    return 0;
}
```

<br>

#### Compile and run

```bash
g++ -std=c++20 -Wall -Wextra -Wpedantic -fsanitize=address \
    -o day52 main.cpp && ./day52
```

<br>

#### Expected output pattern

```
=== UniquePtr (move-only, single-threaded) ===
  Widget 1 ctor
  Widget 1 hello
  Widget 1 hello
  Widget 1 dtor
=== SharedPtr (refcounted, single-threaded) ===
  Widget 2 ctor
  use_count = 1
  use_count = 2
  Widget 2 hello
  use_count = 1
  Widget 2 dtor
=== AtomicSharedPtr (refcounted, multi-threaded counter) ===
  Widget 3 ctor
  use_count = 2
  Widget 3 dtor
=== sizeof: single-threaded vs atomic counter ===
  sizeof(SharedPtr<Widget>)       = 16
  sizeof(AtomicSharedPtr<Widget>) = 16
All done.
```

<br>

#### Bonus Challenges

1. **Break a policy on purpose.** Define `struct BadThreading { using CounterType = int; };` (missing `increment`) and instantiate `SmartPtr<Widget, RefCounted<Widget>, BadThreading>`. Read the concept-driven error — confirm it names the unmet `ThreadingPolicy` requirement at the call site, not deep in the header.

2. **Add a CheckingPolicy axis.** Add a third policy `Checked` vs `Unchecked` that controls `operator*`: `Checked` `assert`s non-null, `Unchecked` does nothing. Constrain it with a concept. Verify the unchecked version inlines to a bare dereference at `-O2`.

3. **`[[no_unique_address]]` for stateful policies.** Make one policy stateful (e.g., a `CountingDeleter` ownership policy that tracks deletions). Store it as a `[[no_unique_address]]` member and show empty policies still cost 0 bytes while the stateful one costs its real size.

4. **Intrusive ownership policy.** Add `IntrusiveRefCounted` where the refcount lives *inside* `T` (T must expose `add_ref`/`release`). Express that requirement as a concept on `T`.

5. **Thread-safety test.** Spin up 8 threads that each copy/destroy an `AtomicSharedPtr` in a loop; run under TSan and confirm no data race, then swap in `SingleThreaded` and watch TSan flag it.

6. **Concept subsumption.** Write two overloads of a free `print(SmartPtr<...>)` — one constrained on a `SharedOwnership` concept, one general — and demonstrate that the more-constrained overload is preferred for shared pointers (subsumption).

<br><br>

---
---

# PART 4: DEEP DIVE — COMMON QUESTIONS

---
---

<br>

<h2 style="color: #2980B9;">📘 52.9 Q&A</h2>

<br>

#### Q1: "How is policy-based design different from the Strategy pattern?"

They solve the same problem — pluggable behavior — at different times. Strategy (Day 19) injects an *object* implementing an interface, dispatched via *virtual* calls, swappable at *runtime*. A policy injects a *type* as a template parameter, dispatched *statically* and inlined, fixed at *compile time*. Policy = zero-overhead, no runtime flexibility; Strategy = runtime flexibility, vtable cost.

<br>

#### Q2: "Inheritance or composition for combining policies?"

Inheritance gets EBO for free and lets the host call policy methods unqualified, but leaks policy members into the host interface and risks name clashes. Composition encapsulates cleanly; pair it with `[[no_unique_address]]` (C++20) to recover EBO. Modern code tends toward composition + `[[no_unique_address]]`.

<br>

#### Q3: "What exactly is a concept — is it a type?"

No. A concept is a named compile-time *predicate over types* (a `constexpr bool`). `ThreadingPolicy<P>` evaluates to `true`/`false`. Concepts are used to *constrain* templates (which types are allowed) and to *order* overloads (more-constrained wins — subsumption). They generate no code themselves.

<br>

#### Q4: "Why are concept error messages so much better than SFINAE?"

A concept failure is reported at the point of use as "constraint X not satisfied because requirement Y failed," before the template body is instantiated. SFINAE failures surface as cryptic substitution errors from deep inside the implementation, often after many frames. Concepts move the check to the boundary and give it a name.

<br>

#### Q5: "Do policies and concepts add runtime cost?"

No. Policies are resolved at compile time and inlined; an empty policy costs zero bytes (EBO / `[[no_unique_address]]`). Concepts are compile-time predicates that vanish after checking. The only "cost" is compile time and potential code bloat (one instantiation per policy combination) — the same trade-off as CRTP (Day 50).

<br>

#### Q6: "When does policy-based design become a liability?"

When the number of policies and their interactions explode, the type names become unwieldy (`SmartPtr<T, RefCounted<T>, MultiThreaded, Checked, ArenaAlloc<T>>`), and template error surfaces grow. Provide *alias templates* (`SharedPtr<T>`) for common combinations, and don't add an axis unless it's genuinely orthogonal. If users need runtime selection, you've outgrown policies — use Strategy.

<br>

#### Q7: "Can a concept constrain relationships between *two* type parameters?"

Yes — a concept can take multiple type parameters: `template<class P, class T> concept OwnershipPolicy = requires(T* p){ ... };` constrains `P` *with respect to* `T`. We used exactly this so the ownership policy's `release` matches the pointee type.

<br>

#### Q8: "How do `requires` clauses interact with overload resolution?"

When several function/class templates match, the one whose constraints *subsume* the others (logically imply them, i.e., are more specific) is chosen. This lets you write a general version and a constrained specialization and have the compiler pick the tightest applicable one — replacing tag dispatch and `enable_if` chains.

<br><br>

---

## Reflection Questions

1. Explain why policy-based design is "compile-time Strategy." What does each gain and give up versus runtime Strategy?
2. What are the two ways a host class can combine policies, and what does `[[no_unique_address]]` add to the composition approach?
3. What makes a policy "good" (orthogonal, minimal interface, ideally stateless)? Give an example of a poorly-factored policy.
4. What is a concept, precisely, and how does it differ from `static_assert` and SFINAE in error quality and overload selection?
5. Name the four kinds of requirement inside a `requires` expression and give an example of each.
6. When would you provide alias templates over a policy-based class, and why?

---

## Interview Questions

1. "What is policy-based design? Contrast it with the Strategy pattern."
2. "Design a smart pointer whose ownership and threading behavior are pluggable policies."
3. "Inheritance vs composition for assembling policies — trade-offs? What does `[[no_unique_address]]` do?"
4. "What is a C++20 concept? Write one that constrains a 'threading policy'."
5. "Show the four kinds of requirement in a `requires` expression."
6. "How do concepts improve on SFINAE and `enable_if` for constraining templates?"
7. "Explain concept subsumption and how it drives overload resolution."
8. "Why do policies add no runtime cost? When *do* they add cost?"
9. "Write a concept that constrains a relationship between two template parameters."
10. "When does policy-based design become unmanageable, and what do you do about it?"

---

**Next**: Day 53 — Mock Interview #1 →
