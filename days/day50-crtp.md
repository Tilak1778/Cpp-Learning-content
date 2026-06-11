# Day 50: CRTP (Curiously Recurring Template Pattern)

[← Back to Study Plan](../lld-study-plan.md) | [← Day 48](day48-shared-memory-ipc.md)

> **Time**: ~1.5-2 hours
> **Goal**: The Curiously Recurring Template Pattern is the idiom where a class derives from a template instantiated with *itself*: `class Derived : public Base<Derived>`. This gives the base class compile-time knowledge of the derived type, enabling **static polymorphism** (dispatch resolved at compile time, no vtable), **mixin** injection of behavior, and zero-overhead interfaces. Learn how `static_cast<Derived*>(this)` works, why it has no runtime cost, the rules and pitfalls (incomplete types, calling derived methods from the base constructor), and exactly when CRTP beats `virtual` and when it doesn't. Build a CRTP logging mixin and a static-interface pattern.

---
---

# PART 1: WHAT IS CRTP AND WHY DOES IT EXIST?

---
---

<br>

<h2 style="color: #2980B9;">📘 50.1 The Core Idea</h2>

Normal inheritance flows "up": a derived class knows about its base. CRTP makes information flow "down" as well — the base is parameterized on the derived type, so **at compile time** the base knows exactly which class derives from it.

```cpp
template <typename Derived>
class Base {
public:
    void interface() {
        // Down-cast to the *concrete* derived type — known at compile time
        static_cast<Derived*>(this)->implementation();
    }
};

class Concrete : public Base<Concrete> {
public:
    void implementation() {
        std::cout << "Concrete::implementation\n";
    }
};
```

The "curious" part is the recursion in the declaration: `Concrete` is defined in terms of `Base<Concrete>`, which mentions `Concrete` before `Concrete` is complete. That's legal — when the compiler instantiates `Base<Concrete>`, it only needs `Concrete`'s *name* (an incomplete type), not its full definition. Member function *bodies* of `Base` are instantiated lazily, only when called, by which point `Concrete` is complete.

```
Virtual dispatch (runtime):                 CRTP dispatch (compile time):

  Base::interface()                            Base<Concrete>::interface()
        │                                              │
        ▼                                              ▼  static_cast — no lookup
  read vptr → vtable → slot → fn ptr          Concrete::implementation()  (inlined)
        │                                              │
        ▼                                              ▼
  indirect CALL (may miss branch predictor)   direct CALL (or fully inlined away)
```

<br>

#### The payoff in one sentence

> CRTP gives you the *polymorphism* of virtual functions (a base defines an algorithm, derived classes fill in the steps) with the *performance* of non-virtual calls (resolved and inlined at compile time, no vtable, no indirection).

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 50.2 Why `static_cast<Derived*>(this)` Is Safe and Free</h2>

Inside `Base<Derived>`, the object `*this` is — by the contract of CRTP — *actually* a `Derived`. The `static_cast` simply tells the compiler "treat this `Base<Derived>*` as the `Derived*` it really is."

- **Safe**: as long as the only thing that derives from `Base<Derived>` is `Derived` itself. The cast is undefined behavior if some *other* class `Evil : public Base<Derived>` exists, because then `this` is not a `Derived`. (We'll see how to forbid that in §50.6.)
- **Free**: a `static_cast` between a base and derived pointer is, in the single-inheritance non-virtual case, either a no-op or a fixed compile-time pointer offset. There is **no runtime type check** (unlike `dynamic_cast`) and no memory read (unlike a vtable lookup).

```cpp
template <typename D>
class Base {
protected:
    // Convenience accessors — used pervasively in CRTP code
    D&       self()       { return static_cast<D&>(*this); }
    const D& self() const { return static_cast<const D&>(*this); }
};
```

Because the call target is known statically, the optimizer can **inline** `self().implementation()` straight into `interface()`. The whole abstraction frequently compiles down to nothing.

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 50.3 The Three Uses of CRTP</h2>

CRTP shows up in three distinct roles. Keep them separate in your head — interviewers probe whether you understand *why* you'd reach for it.

| Use | What the base provides | Example |
|-----|------------------------|---------|
| **Static interface / static polymorphism** | A templated algorithm that calls into derived "hook" methods, dispatched at compile time | A `Shape<T>` base whose `area()` calls `T::compute_area()` |
| **Mixin (injecting behavior)** | Whole pieces of functionality "mixed into" the derived class for free | `Comparable<T>`, `Printable<T>`, `Counter<T>`, `Cloneable<T>` |
| **Static polymorphic container / dispatch** | A common entry point that fans out to many derived types without virtual calls | Expression templates, `enable_shared_from_this` |

The mixin use is the most common in everyday code. The static-interface use is the one most often asked about in interviews ("how do you get polymorphism without virtual?").

<br><br>

---
---

# PART 2: CRTP VS VIRTUAL — THE REAL TRADE-OFF

---
---

<br>

<h2 style="color: #2980B9;">📘 50.4 Side-by-Side</h2>

The same "template method" design, two ways:

```cpp
// ---- Runtime polymorphism (virtual) ----
struct AnimalV {
    virtual ~AnimalV() = default;
    virtual std::string sound() const = 0;
    void describe() const {                    // template method
        std::cout << "I say " << sound() << "\n";
    }
};
struct DogV : AnimalV { std::string sound() const override { return "woof"; } };

// ---- Compile-time polymorphism (CRTP) ----
template <typename D>
struct AnimalC {
    void describe() const {                    // template method
        std::cout << "I say " << self().sound() << "\n";
    }
    const D& self() const { return static_cast<const D&>(*this); }
};
struct DogC : AnimalC<DogC> { std::string sound() const { return "woof"; } };
```

Both let `describe()` call a derived-supplied `sound()`. The difference is *when* and *how* the call to `sound()` is resolved.

<br>

<h2 style="color: #2980B9;">📘 50.5 The Decision Table</h2>

| Dimension | `virtual` (runtime) | CRTP (compile time) |
|-----------|---------------------|---------------------|
| **Dispatch cost** | Indirect call via vtable; possible branch misprediction & i-cache miss | Direct call, usually inlined to zero |
| **Object size** | +1 vptr (8 bytes) per object | No vptr; potentially empty base (EBO) |
| **Heterogeneous container** | ✅ `vector<unique_ptr<Animal>>` works | ❌ Each `Animal<D>` is a *different type*; cannot store mixed in one vector |
| **Decide type at runtime** | ✅ Choose subclass from input/config | ❌ Type fixed at compile time |
| **Binary / ABI stability** | Stable interface across translation units | Templates leak into headers; recompiles ripple |
| **Compile time & code bloat** | One copy of the function | One instantiation *per* derived type |
| **Error messages** | Clean | Notoriously verbose template errors |
| **Separate compilation** | `.cpp` can hide implementation | Must live in headers |

<br>

#### The one-line rule

> Use **CRTP** when the set of types is known at compile time and you need the call to be free (hot loops, libraries, mixins). Use **`virtual`** when you need a heterogeneous collection, runtime type selection, or stable binary interfaces.

If you ever find yourself writing `std::vector<Base*>` and pushing different CRTP-derived types into it, you've picked the wrong tool — CRTP cannot do that without a *second* layer of type erasure on top (which is Day 51's topic).

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 50.6 Hardening CRTP — Forbidding the "Wrong Derived"</h2>

The static_cast in the base is undefined behavior if a class derives from `Base<X>` but is not `X`:

```cpp
class A : public Base<A> {};
class B : public Base<A> {};   // ☠️ B claims to be A — static_cast<A*>(this) is UB
```

Two standard defenses:

```cpp
template <typename D>
class Base {
private:
    // 1. Private constructor + friend: only D can construct Base<D>,
    //    so only D can derive from it.
    Base() = default;
    friend D;

protected:
    D& self() { return static_cast<D&>(*this); }
};
```

```cpp
// 2. A static_assert in a member (checked when instantiated):
template <typename D>
void Base<D>::interface() {
    static_assert(std::is_base_of_v<Base<D>, D>,
                  "D must derive from Base<D>");
    self().impl();
}
```

The `private constructor + friend D` trick is the elegant one: `B : public Base<A>` won't compile because `B` is not a friend of `Base<A>` and therefore cannot call its constructor.

<br><br>

---
---

# PART 3: MIXINS — STACKING BEHAVIOR

---
---

<br>

<h2 style="color: #2980B9;">📘 50.7 The Mixin Idea</h2>

A *mixin* is a small CRTP base that grants a derived class a self-contained capability, deriving that capability *from* the derived class's own members. The classic example is generating comparison operators from a single `compare()`:

```cpp
template <typename D>
struct Comparable {
    friend bool operator==(const D& a, const D& b) { return a.compare(b) == 0; }
    friend bool operator!=(const D& a, const D& b) { return a.compare(b) != 0; }
    friend bool operator< (const D& a, const D& b) { return a.compare(b) <  0; }
    friend bool operator> (const D& a, const D& b) { return a.compare(b) >  0; }
    friend bool operator<=(const D& a, const D& b) { return a.compare(b) <= 0; }
    friend bool operator>=(const D& a, const D& b) { return a.compare(b) >= 0; }
};

struct Version : Comparable<Version> {
    int major, minor;
    int compare(const Version& o) const {
        if (major != o.major) return major - o.major;
        return minor - o.minor;
    }
};
// Version now has all six comparison operators, derived from one method.
```

> Note: in C++20 the spaceship operator `operator<=>` makes *this particular* mixin obsolete — but the *technique* generalizes to any behavior (cloning, counting, serialization) where there's no language feature to lean on.

<br>

#### Stacking multiple mixins

Because each mixin is a separate base, you can compose them:

```cpp
struct Widget
    : Comparable<Widget>,
      Printable<Widget>,
      InstanceCounter<Widget> {
    /* ... */
};
```

This is *much* lighter than virtual interfaces: each mixin adds **no vptr** (often an empty base, so 0 bytes via the Empty Base Optimization) and **no runtime cost**.

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 50.8 The Empty Base Optimization (EBO)</h2>

A mixin with no data members is an *empty* class. C++ guarantees that an empty base subobject can occupy **zero bytes** when used as a base (it can't when used as a member — every member needs a distinct address).

```cpp
struct Empty {};
struct AsMember  { Empty e; int x; };   // sizeof >= 8 (e takes a byte + padding)
struct AsBase : Empty { int x; };       // sizeof == 4 (Empty contributes 0)
```

This is *why* CRTP mixins are free in space: `struct Version : Comparable<Version>` is the same size as a bare `Version`. Virtual interfaces can't do this — they always add a vptr.

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 50.9 The `enable_shared_from_this` Connection</h2>

You've already used CRTP if you've used `std::enable_shared_from_this`:

```cpp
class Session : public std::enable_shared_from_this<Session> {
    void start() {
        auto self = shared_from_this();   // hand a shared_ptr to an async callback
        async_read([self]{ /* keeps Session alive */ });
    }
};
```

`enable_shared_from_this<Session>` is a CRTP base. It stores a `weak_ptr<Session>` that the `shared_ptr` machinery populates, and `shared_from_this()` returns a `shared_ptr<Session>` — typed exactly to the derived class because the base was parameterized on it. (See Day 3, `day03-shared-ptr.md`, for the shared_ptr internals this builds on.)

<br><br>

---
---

# PART 4: PITFALLS

---
---

<br>

<h2 style="color: #2980B9;">📘 50.10 Common Mistakes</h2>

#### Pitfall 1: Calling a derived method from the base *constructor*

```cpp
template <typename D>
struct Base {
    Base() { static_cast<D*>(this)->init(); }   // ☠️ D part not constructed yet
};
```

During `Base<D>`'s constructor, the `D` subobject hasn't been constructed (base ctors run first). Calling `D::init()` touches uninitialized members → UB. With *virtual* inheritance you'd "just" get the base's version; with CRTP you get genuine UB. **Never call `self().anything()` from the CRTP base constructor or destructor.**

<br>

#### Pitfall 2: Forgetting the derived method exists

```cpp
template <typename D>
struct Base { void run() { self().step(); } };
struct C : Base<C> { /* forgot to define step() */ };
// Error only appears when run() is instantiated — and the message is ugly.
```

CRTP "interfaces" aren't checked at the point of derivation; they're checked when the base method is instantiated. C++20 **Concepts** (Day 52) let you state the requirement up front for a clean diagnostic.

<br>

#### Pitfall 3: Name hiding / accidental recursion

If both base and derived define `run()`, the base's `self().run()` may resolve to the derived `run()`, which calls `self().run()`... infinite recursion. Use distinct names for the hook (`step`, `impl`) vs the public entry point (`run`).

<br>

#### Pitfall 4: Trying to store CRTP objects polymorphically

```cpp
std::vector<Base*> v;        // Base is a template — Base<A>* and Base<B>* are unrelated
```

There is no common base type. If you need a heterogeneous container, CRTP is the wrong tool — reach for `virtual` or type erasure (Day 51).

<br>

#### Pitfall 5: Code bloat

Every distinct `Derived` produces a fresh instantiation of every base method it uses. With dozens of derived types and large methods, this can balloon binary size and compile time. Profile before assuming "compile-time = always better."

<br><br>

---
---

# PART 5: BUILD EXERCISE

---
---

<br>

<h2 style="color: #2980B9;">📘 50.11 Exercise: A CRTP Logging Mixin + Static Interface</h2>

**Motivation.** We want two things that mainstream OOP would solve with virtual functions, but without paying for a vtable:

1. A **logging mixin** `LogMixin<D>` that gives any class a `log(msg)` method which automatically prefixes output with the derived class's *name* and a per-class instance counter — behavior injected for free.
2. A **static interface** `Shape<D>` that defines a `describe()` template method calling derived hooks `name()` and `area()`, dispatched entirely at compile time.

This demonstrates both major CRTP roles in one program, and shows the EBO (the mixin adds no size).

<br>

#### Skeleton

```cpp
// crtp.h
#pragma once
#include <iostream>
#include <string>
#include <cmath>

// ─────────────────────────────────────────────────────────────
// 1) MIXIN: injects a log() method + per-derived instance counter.
//    Empty-ish base so it adds (near) zero size via EBO.
// ─────────────────────────────────────────────────────────────
template <typename Derived>
class LogMixin {
public:
    void log(const std::string& msg) const {
        // self().tag() is a derived-provided hook resolved at compile time.
        std::cout << "[" << self().tag() << " #" << self().id() << "] "
                  << msg << "\n";
    }

    // Per-derived-type counter: each Derived has its OWN static count,
    // because LogMixin<Derived> is a distinct type per Derived.
    static int live_count() { return s_count; }

protected:
    LogMixin()  { ++s_count; }            // safe: no call into Derived here
    ~LogMixin() { --s_count; }
    LogMixin(const LogMixin&) { ++s_count; }

private:
    const Derived& self() const { return static_cast<const Derived&>(*this); }
    static inline int s_count = 0;        // C++17 inline static — one per Derived
};

// ─────────────────────────────────────────────────────────────
// 2) STATIC INTERFACE: a template method that dispatches to hooks
//    name() and area() at compile time. No virtual, no vtable.
// ─────────────────────────────────────────────────────────────
template <typename Derived>
class Shape {
public:
    void describe() const {
        std::cout << self().name() << " has area " << self().area() << "\n";
    }
    bool bigger_than(const Shape& other) const {
        return self().area() > other.self().area();   // same-type comparison
    }

private:
    // Hardening: only Derived can construct Shape<Derived> (see §50.6).
    Shape() = default;
    friend Derived;

    const Derived& self() const { return static_cast<const Derived&>(*this); }
};
```

```cpp
// concrete types: each uses BOTH the mixin and the static interface
#include "crtp.h"

class Circle : public Shape<Circle>, public LogMixin<Circle> {
    double r_;
    int    id_;
public:
    explicit Circle(double r) : r_(r), id_(LogMixin::live_count()) {}

    // Shape hooks:
    std::string name() const { return "Circle"; }
    double      area() const { return M_PI * r_ * r_; }

    // LogMixin hooks:
    std::string tag()  const { return "Circle"; }
    int         id()   const { return id_; }
};

class Square : public Shape<Square>, public LogMixin<Square> {
    double s_;
    int    id_;
public:
    explicit Square(double s) : s_(s), id_(LogMixin::live_count()) {}

    std::string name() const { return "Square"; }
    double      area() const { return s_ * s_; }

    std::string tag()  const { return "Square"; }
    int         id()   const { return id_; }
};
```

<br>

#### Test driver

```cpp
// main.cpp
#include "crtp.h"
#include <cassert>

// Re-declare the concrete classes here or #include them; inlined for brevity.
class Circle; class Square;   // (in a real build, put them in a header)

int main() {
    std::cout << "=== sizeof check: mixin should add ~0 via EBO ===\n";
    struct Bare { double r; int id; };
    std::cout << "  sizeof(Bare)   = " << sizeof(Bare)   << "\n";
    std::cout << "  sizeof(Circle) = " << sizeof(Circle) << "\n";
    // Circle adds two empty CRTP bases -> same data size as Bare (+ maybe 1 byte).

    std::cout << "\n=== static interface (compile-time dispatch) ===\n";
    Circle c1(2.0);
    Square s1(3.0);
    c1.describe();                         // "Circle has area 12.566..."
    s1.describe();                         // "Square has area 9"
    std::cout << "  circle bigger than square? "
              << std::boolalpha << c1.bigger_than(... ) << "\n";  // see note below

    std::cout << "\n=== logging mixin ===\n";
    c1.log("created");                     // "[Circle #0] created"
    s1.log("ready");                       // "[Square #0] ready"

    std::cout << "\n=== per-derived instance counters ===\n";
    Circle c2(1.0), c3(1.0);
    std::cout << "  live Circles = " << Circle::live_count() << "\n";  // 3
    std::cout << "  live Squares = " << Square::live_count() << "\n";  // 1

    std::cout << "\nAll done.\n";
    return 0;
}
```

> Implementation note for the learner: `bigger_than` compares two *same-typed* shapes. To compare a `Circle` against a `Square` you would have to fall back to runtime polymorphism or compare `area()` values directly — a perfect illustration of CRTP's heterogeneity limitation. For the build, change `bigger_than` to take a `double` area, or compare two circles. The point is to *feel* the constraint.

<br>

#### Compile and run

```bash
g++ -std=c++17 -Wall -Wextra -Wpedantic -fsanitize=address \
    -o day50 main.cpp && ./day50
```

<br>

#### Expected output pattern

```
=== sizeof check: mixin should add ~0 via EBO ===
  sizeof(Bare)   = 16
  sizeof(Circle) = 16

=== static interface (compile-time dispatch) ===
Circle has area 12.5664
Square has area 9

=== logging mixin ===
[Circle #0] created
[Square #0] ready

=== per-derived instance counters ===
  live Circles = 3
  live Squares = 1

All done.
```

<br>

#### Bonus Challenges

1. **Harden against the wrong derived.** Add `static_assert(std::is_base_of_v<Shape<Circle>, Circle>)` inside `describe()` and then try to write `struct Evil : Shape<Circle> {}` — observe whether the `private ctor + friend Derived` trick already blocks it at compile time.

2. **Add a `clone()` mixin.** Write `template<class D> struct Cloneable` whose `clone()` returns `D(static_cast<const D&>(*this))` by value, with no virtual. Discuss why this can't return a polymorphic `Base*`.

3. **Prove the inlining.** Compile with `-O2 -S` and inspect the assembly for `describe()` / `bigger_than()`. Confirm there is no `call` to `area()` — it's inlined. Compare against the virtual version.

4. **Measure the size win.** Build a virtual `Shape` hierarchy with the same data and `sizeof` both. Show the vptr costs 8 bytes per object that CRTP avoids.

5. **Static visitor.** Write a free function `template<class D> void render(const Shape<D>& s)` that works for any shape via static dispatch. Note how it differs from a runtime visitor.

6. **Mixin ordering & EBO stress test.** Stack three empty mixins on one class and verify with `sizeof` that all three contribute zero bytes. Then make one mixin hold a member and observe the size change.

<br><br>

---
---

# PART 6: DEEP DIVE — COMMON QUESTIONS

---
---

<br>

<h2 style="color: #2980B9;">📘 50.12 Q&A</h2>

<br>

#### Q1: "Why is it called *curiously recurring*?"

The name comes from James Coplien (1995), who noticed the pattern "recurring" in template code: a class `X` appearing as the template argument to its own base, `class X : Base<X>`. The self-reference looked curious because, at the point of declaration, `X` is still incomplete.

<br>

#### Q2: "How can `Base<Derived>` mention `Derived` before it's defined?"

Class templates only need the *declared name* of `Derived` (an incomplete type) to form `Base<Derived>`. Member function *bodies* aren't instantiated until used, by which time `Derived` is complete. That's why you can't dereference or `sizeof` `Derived` in the base class body at namespace scope — but you can inside a member function that's instantiated later.

<br>

#### Q3: "Is the `static_cast` ever actually a runtime operation?"

In single inheritance with non-virtual bases (the normal CRTP case), no — it's a compile-time pointer adjustment, often the identity. It becomes a non-trivial offset only with multiple inheritance, and it is *never* a runtime check (that's `dynamic_cast`). It compiles to nothing or a constant add.

<br>

#### Q4: "When is CRTP strictly worse than virtual?"

When you need a heterogeneous container, runtime type selection (e.g., choosing a subclass from a config string), a stable binary interface across `.so` boundaries, or fast compile times with many derived types. In those cases the vtable's indirection is a feature, not a bug.

<br>

#### Q5: "Can I combine CRTP and virtual?"

Yes — a common library trick is a virtual base interface plus a CRTP helper that *implements* the boilerplate. E.g., `enable_shared_from_this` is CRTP, but the objects can still be polymorphic via a separate virtual base. They solve different problems and compose fine.

<br>

#### Q6: "Why does each derived type get its own static member in the mixin?"

Because `LogMixin<Circle>` and `LogMixin<Square>` are **different instantiations** — different types — each with its own copy of any `static` data member. That's exactly what makes a per-type instance counter trivial in CRTP: no manual registry needed.

<br>

#### Q7: "What's the relationship between CRTP and the 'static interface' / 'duck typing in templates'?"

A plain function template `template<class T> void f(T& t){ t.foo(); }` already does compile-time duck typing. CRTP adds *structure*: the base provides a reusable algorithm (template method), and inheritance documents the contract. Concepts (Day 52) make that contract explicit and enforceable.

<br>

#### Q8: "Does CRTP break with multiple inheritance from several mixins?"

No — that's the intended use (stacking mixins). Each mixin is parameterized on the *final* derived type, so all of them can `static_cast` to it. Just avoid name collisions among the mixins' members and don't let two mixins both define the same hook the base expects.

<br><br>

---

## Reflection Questions

1. Explain in one sentence how CRTP achieves polymorphism without a vtable. What does `static_cast<Derived*>(this)` compile to?
2. Why is calling a derived method from the CRTP base *constructor* undefined behavior, and how would you restructure to avoid it?
3. What is the Empty Base Optimization and why is it central to why CRTP mixins are "free"?
4. Give a concrete scenario where CRTP is the *wrong* choice and `virtual` is required. Why?
5. How does the `private constructor + friend Derived` idiom prevent the "wrong derived" undefined behavior?
6. Why does each `Derived` get its own copy of a `static` member declared in the CRTP base?

---

## Interview Questions

1. "Implement static polymorphism with CRTP: a base `describe()` that calls derived `name()`/`area()`."
2. "Compare CRTP and virtual functions across dispatch cost, object size, and heterogeneous containers."
3. "How does `std::enable_shared_from_this` use CRTP?"
4. "Write a `Comparable<T>` mixin that synthesizes all six comparison operators from one `compare()`."
5. "Why can `Base<Derived>` reference `Derived` before `Derived` is defined?"
6. "What goes wrong if a class derives from `Base<SomeOtherType>`? How do you prevent it?"
7. "Show how CRTP avoids the per-object vtable pointer, and prove the size difference with `sizeof`."
8. "When would you *not* use CRTP despite its performance advantage?"
9. "Explain code bloat as a downside of CRTP. How would you measure it?"
10. "How do C++20 concepts improve CRTP-based static interfaces?"

---

**Next**: Day 51 — Type Erasure →
