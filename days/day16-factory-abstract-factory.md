# Day 16: Factory & Abstract Factory

[← Back to Study Plan](../lld-study-plan.md) | [← Day 15](day15-singleton.md)

> **Time**: ~1.5-2 hours
> **Goal**: Understand the Factory family of creational patterns — Simple Factory, Factory Method, Abstract Factory, and the **registration-based factory** that powers most real-world plugin systems. Learn why "decouple creation from usage" matters, how to return ownership safely with `unique_ptr`, and how self-registering creators let you add new types without touching existing code. Build a Shape Factory that creates objects from a string name.

---
---

# PART 1: WHAT IS A FACTORY AND WHY DOES IT EXIST?

---
---

<br>

<h2 style="color: #2980B9;">📘 16.1 The Core Idea</h2>

A factory **encapsulates object creation**. Instead of constructing objects directly with `new` at every call site, you ask a factory to produce them.

```
Without a factory:                        With a factory:
  Shape* s;                                 Shape* s = ShapeFactory::create("circle");
  if (type == "circle")        →
      s = new Circle();
  else if (type == "square")
      s = new Square();
  else if (type == "triangle")
      s = new Triangle();
  ...
```

The caller no longer:
- Knows the concrete class names
- Knows which header to include
- Cares how the object is constructed
- Has to be modified when a new shape is added

The caller speaks the **abstract** language ("give me a shape called X"); the factory speaks the **concrete** language (`new Circle`).

<br>

#### What does "decouple creation from usage" actually mean?

| | Tight coupling (no factory) | Loose coupling (factory) |
|---|---------------------------|--------------------------|
| **Headers** | Every caller `#include`s every concrete class | Callers `#include` only the abstract base |
| **Compile time** | One change to a concrete class rebuilds everything | Only the factory's TU rebuilds |
| **Adding a new type** | Edit every `if/else` chain that picks types | Register the new type in one place |
| **Testing** | Hardcoded `new` calls — can't substitute | Replace the factory with a mock |
| **Runtime selection** | Painful — caller must know all types | Trivial — string/enum keys |

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 16.2 The Symptoms of "Just Use new"</h2>

Direct construction looks innocent until you have **branching logic** or **plugin-style extensibility**.

<br>

#### Symptom 1: The conditional explosion

```cpp
// Caller code, scattered across many files
Shape* makeShape(const std::string& kind) {
    if (kind == "circle")    return new Circle();
    if (kind == "square")    return new Square();
    if (kind == "triangle")  return new Triangle();
    if (kind == "hexagon")   return new Hexagon();
    if (kind == "ellipse")   return new Ellipse();
    // ... 30 more lines
    throw std::runtime_error("unknown shape");
}
```

Every new shape forces a code change in this function — and probably in **every other function** that switches on shape type. This violates the **Open/Closed Principle** (open for extension, closed for modification).

<br>

#### Symptom 2: Ownership ambiguity

```cpp
Shape* s = new Circle();
// Who deletes it? When? On which path?
// Did the caller remember to wrap it in a smart pointer?
```

A factory that always returns `std::unique_ptr<Shape>` makes ownership unambiguous and eliminates leaks.

<br>

#### Symptom 3: Construction logic leaking everywhere

```cpp
// Constructing a Connection requires reading config, opening a socket,
// authenticating, allocating buffers... this complexity is now duplicated:
Connection* a = new Connection(host, port, ssl, timeout, /*lots more*/);
Connection* b = new Connection(host, port, ssl, timeout, /*lots more*/);
```

Centralize this in `ConnectionFactory::create()` and the construction recipe lives in **one** place.

<br>

**The factory pattern's job**: lift creation out of caller code into a single dedicated boundary, so callers see only abstractions.

<br><br>

---
---

# PART 2: FACTORY VARIANTS

---
---

<br>

<h2 style="color: #2980B9;">📘 16.3 Variant 1: Simple Factory (a.k.a. Static Factory Method)</h2>

A class with a static method that switches on a type identifier. **Not technically a GoF pattern**, but the most common starting point.

```cpp
class Shape {
public:
    virtual ~Shape() = default;
    virtual void draw() const = 0;
};

class Circle   : public Shape { public: void draw() const override { std::cout << "○\n"; } };
class Square   : public Shape { public: void draw() const override { std::cout << "□\n"; } };
class Triangle : public Shape { public: void draw() const override { std::cout << "△\n"; } };

class ShapeFactory {
public:
    static std::unique_ptr<Shape> create(const std::string& kind) {
        if (kind == "circle")   return std::make_unique<Circle>();
        if (kind == "square")   return std::make_unique<Square>();
        if (kind == "triangle") return std::make_unique<Triangle>();
        return nullptr;
    }
};

// Usage:
auto s = ShapeFactory::create("circle");
s->draw();
```

<br>

#### Pros and cons

| Pros | Cons |
|------|------|
| Trivial to write | Adding a shape requires editing `ShapeFactory::create` |
| One place to look for "how is X built?" | Violates open/closed principle |
| Hides concrete types from callers | The factory itself depends on every concrete class |

This is fine for a **closed set** of types known at compile time. It fails when you want plugins or third-party extensions.

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 16.4 Variant 2: Factory Method (Polymorphic Creation)</h2>

A *base class* declares a virtual creation method; *derived classes* override it to produce different concrete products. The factory is the object itself.

```cpp
class Document {
public:
    virtual ~Document() = default;
    virtual std::string render() const = 0;
};

class HtmlDoc  : public Document { public: std::string render() const override { return "<html>...</html>"; } };
class PdfDoc   : public Document { public: std::string render() const override { return "%PDF-1.4..."; } };

class Application {
public:
    virtual ~Application() = default;

    // Factory method — subclasses pick the concrete Document
    virtual std::unique_ptr<Document> createDocument() = 0;

    // Template method that USES the factory method
    void newFile() {
        auto doc = createDocument();
        std::cout << doc->render() << "\n";
    }
};

class WebApplication : public Application {
public:
    std::unique_ptr<Document> createDocument() override {
        return std::make_unique<HtmlDoc>();
    }
};

class DesktopApplication : public Application {
public:
    std::unique_ptr<Document> createDocument() override {
        return std::make_unique<PdfDoc>();
    }
};

// Usage:
WebApplication web;
web.newFile();   // prints HTML

DesktopApplication desktop;
desktop.newFile();  // prints PDF
```

<br>

The key idea: **the parent class's algorithm calls the factory method**, and subclasses customize *what* gets created without rewriting the algorithm. This is the Factory Method pattern as defined in GoF.

<br>

#### When to use Factory Method

- A framework defines a workflow but lets users pick the products (e.g., MFC, Qt's `QApplication` subclassing)
- You want to apply the **template method pattern** with a customization hook for object creation
- The factory and product classes are paired in a parallel hierarchy

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 16.5 Variant 3: Abstract Factory (Families of Products)</h2>

When you need to create **families of related products** — and a client must use products from the **same family** consistently — Abstract Factory is the answer.

```
Theme (family): Dark           Light
Products:        DarkButton    LightButton
                 DarkScrollbar LightScrollbar
                 DarkMenu      LightMenu

Rule: never mix a DarkButton with a LightScrollbar.
```

<br>

#### Implementation

```cpp
// Abstract products
class Button    { public: virtual ~Button() = default;    virtual void paint() const = 0; };
class Scrollbar { public: virtual ~Scrollbar() = default; virtual void paint() const = 0; };

// Dark family
class DarkButton    : public Button    { public: void paint() const override { std::cout << "[Dark Button]\n";    } };
class DarkScrollbar : public Scrollbar { public: void paint() const override { std::cout << "[Dark Scrollbar]\n"; } };

// Light family
class LightButton    : public Button    { public: void paint() const override { std::cout << "[Light Button]\n";    } };
class LightScrollbar : public Scrollbar { public: void paint() const override { std::cout << "[Light Scrollbar]\n"; } };

// Abstract factory — declares creators for each product in the family
class ThemeFactory {
public:
    virtual ~ThemeFactory() = default;
    virtual std::unique_ptr<Button>    createButton()    = 0;
    virtual std::unique_ptr<Scrollbar> createScrollbar() = 0;
};

// Concrete factories — one per family
class DarkThemeFactory : public ThemeFactory {
public:
    std::unique_ptr<Button>    createButton()    override { return std::make_unique<DarkButton>(); }
    std::unique_ptr<Scrollbar> createScrollbar() override { return std::make_unique<DarkScrollbar>(); }
};

class LightThemeFactory : public ThemeFactory {
public:
    std::unique_ptr<Button>    createButton()    override { return std::make_unique<LightButton>(); }
    std::unique_ptr<Scrollbar> createScrollbar() override { return std::make_unique<LightScrollbar>(); }
};

// Client — works only with abstractions
void renderUI(ThemeFactory& factory) {
    auto button    = factory.createButton();
    auto scrollbar = factory.createScrollbar();
    button->paint();
    scrollbar->paint();
}

// Usage:
DarkThemeFactory darkFactory;
renderUI(darkFactory);

LightThemeFactory lightFactory;
renderUI(lightFactory);
```

<br>

#### Factory Method vs Abstract Factory — easy to confuse

| | Factory Method | Abstract Factory |
|---|---------------|------------------|
| **Granularity** | Creates **one** product | Creates a **family** of related products |
| **Extension mechanism** | Inheritance — override the method | Composition — supply a different factory object |
| **Typical signature** | `virtual Product* create()` (single method) | Multiple methods on the factory class |
| **Picture** | "What document does this app open?" | "What button + scrollbar + menu look go together?" |

A common pattern: an Abstract Factory's individual creator methods are themselves Factory Methods. They're complementary, not competing.

<br><br>

---
---

# PART 3: REGISTRATION-BASED FACTORY

---
---

<br>

<h2 style="color: #2980B9;">📘 16.6 The Open/Closed Problem with Switch-Based Factories</h2>

Look again at Simple Factory:

```cpp
static std::unique_ptr<Shape> create(const std::string& kind) {
    if (kind == "circle")   return std::make_unique<Circle>();
    if (kind == "square")   return std::make_unique<Square>();
    // To add Hexagon, you must EDIT this function.
}
```

If `Shape` is part of a public library and `Hexagon` is defined by a **plugin** that you can't recompile against, this approach is broken.

We need: **add new types from outside the factory's translation unit, with no source change to the factory.**

The solution: **store creators in a registry** keyed by string. The factory becomes a lookup.

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 16.7 Registration-Based Factory — Implementation</h2>

```cpp
#include <functional>
#include <memory>
#include <string>
#include <unordered_map>

class Shape {
public:
    virtual ~Shape() = default;
    virtual void draw() const = 0;
};

class ShapeFactory {
public:
    using Creator = std::function<std::unique_ptr<Shape>()>;

    // Register a creator under a name
    static bool registerShape(const std::string& name, Creator creator) {
        return registry().emplace(name, std::move(creator)).second;
    }

    // Create by name; returns nullptr if not registered
    static std::unique_ptr<Shape> create(const std::string& name) {
        auto it = registry().find(name);
        if (it == registry().end()) return nullptr;
        return (it->second)();
    }

    // Introspection — list registered names
    static std::vector<std::string> registered() {
        std::vector<std::string> names;
        for (const auto& [name, _] : registry()) names.push_back(name);
        return names;
    }

private:
    // Meyer's-singleton registry — avoids static-init-order fiasco
    static std::unordered_map<std::string, Creator>& registry() {
        static std::unordered_map<std::string, Creator> instance;
        return instance;
    }
};
```

<br>

#### Why a function inside a static getter, not a static member?

```cpp
static std::unordered_map<std::string, Creator> s_registry;  // BAD
```

Concrete classes self-register at static-initialization time (see 16.8). The order of static initialization across translation units is **undefined**. If a concrete class's registration runs before `s_registry` is constructed, you get undefined behavior — inserting into an unconstructed map.

The Meyer's-singleton trick (`static map& registry()`) constructs the map on first access. Whichever caller wins the race triggers construction; everyone else sees the constructed map. This is exactly the same fix you applied for singletons on Day 15.

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 16.8 Self-Registration via Static Initializer</h2>

The trick: each concrete class triggers its own registration at program startup, using a **static variable whose initializer calls `registerShape`**.

```cpp
// circle.cpp
class Circle : public Shape {
public:
    void draw() const override { std::cout << "○\n"; }
};

// The "registrar" — initialized at static init time, calls registerShape
static const bool circle_registered =
    ShapeFactory::registerShape("circle",
        []() -> std::unique_ptr<Shape> { return std::make_unique<Circle>(); });
```

```cpp
// square.cpp
class Square : public Shape {
public:
    void draw() const override { std::cout << "□\n"; }
};

static const bool square_registered =
    ShapeFactory::registerShape("square",
        []() -> std::unique_ptr<Shape> { return std::make_unique<Square>(); });
```

<br>

#### How this enables the Open/Closed Principle

```
Adding a Hexagon:
  1. Create hexagon.cpp with class Hexagon
  2. Add a `static const bool` registrar
  3. Compile and link

Files modified in the factory or in any other shape's code: ZERO.
```

The factory has no knowledge of `Hexagon`. The caller can write `ShapeFactory::create("hexagon")` and it just works.

<br>

#### A clean macro to reduce boilerplate

```cpp
#define REGISTER_SHAPE(name, ClassType)                                  \
    static const bool ClassType##_registered =                            \
        ShapeFactory::registerShape(name,                                 \
            []() -> std::unique_ptr<Shape> {                              \
                return std::make_unique<ClassType>();                     \
            })

// Usage:
REGISTER_SHAPE("circle",   Circle);
REGISTER_SHAPE("square",   Square);
REGISTER_SHAPE("triangle", Triangle);
```

<br>

#### Beware: static linker stripping

If a concrete class lives in a **static library** (`.a` / `.lib`) and the main program never references its symbols directly, the linker may **drop the entire object file** — taking the registrar with it. Result: `create("circle")` returns `nullptr` because the registration never ran.

Fixes:
- Compile concrete classes into a **shared library** (`.so` / `.dll`) and load it
- Use `-Wl,--whole-archive` (GCC) or `/WHOLEARCHIVE` (MSVC) when linking
- Add a dummy reference to each concrete class from the main TU

<br><br>

---
---

# PART 4: OWNERSHIP & MODERN C++

---
---

<br>

<h2 style="color: #2980B9;">📘 16.9 Always Return <code>std::unique_ptr</code></h2>

A factory that returns a raw pointer is a leak waiting to happen.

```cpp
// BAD — caller must remember to delete
Shape* create(const std::string& name);

auto* s = ShapeFactory::create("circle");
// forgot to delete? leak.
// returned early via exception? leak.

// GOOD — ownership is unambiguous
std::unique_ptr<Shape> create(const std::string& name);

auto s = ShapeFactory::create("circle");
// destructor runs automatically when s goes out of scope
```

<br>

#### Why `unique_ptr<Base>` and not `unique_ptr<Derived>`?

The factory's signature is the **public contract**. Callers should see only the abstract base. `unique_ptr<Derived>` leaks implementation details.

```cpp
std::unique_ptr<Shape> ShapeFactory::create(const std::string& name) {
    return std::make_unique<Circle>();  // converts to unique_ptr<Shape> automatically
}
```

This works because `unique_ptr<Derived>` is implicitly convertible to `unique_ptr<Base>` when `Base` has a virtual destructor (which yours must — you delete derived objects through a base pointer).

<br>

#### When to return `shared_ptr`

Return `unique_ptr` by default; let the caller `std::move` it into a `shared_ptr` if they need shared ownership. Returning `shared_ptr` from a factory forces every caller to pay the control-block cost even when they only need exclusive ownership.

```cpp
auto unique = ShapeFactory::create("circle");
std::shared_ptr<Shape> shared = std::move(unique);  // upgrade if needed
```

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 16.10 Lambda Creators vs Function Pointers vs Templates</h2>

Three ways to store a creator. Trade-offs:

<br>

#### A. `std::function<unique_ptr<Shape>()>` — most flexible

```cpp
using Creator = std::function<std::unique_ptr<Shape>()>;
```

- ✅ Captures arbitrary state (lambda captures, member function bindings)
- ✅ Easy to write
- ❌ Type erasure → small heap allocation + virtual dispatch on each call
- ❌ Slightly larger object size

This is the right default. Don't optimize the creator call — it's amortized over object construction cost.

<br>

#### B. Plain function pointer — fastest, least flexible

```cpp
using Creator = std::unique_ptr<Shape>(*)();
```

- ✅ Smallest, fastest
- ❌ Can't capture state — registrar must be a free function or stateless lambda

Acceptable when you know creators have no state, but `std::function` is rarely a bottleneck.

<br>

#### C. Templated factory — compile-time creators

```cpp
template<typename T>
std::unique_ptr<Shape> defaultCreator() {
    return std::make_unique<T>();
}

ShapeFactory::registerShape("circle", &defaultCreator<Circle>);
```

- ✅ Zero-overhead direct call
- ✅ Reusable
- ❌ Each `T` instantiates a new creator function

<br>

**Recommendation**: stick with `std::function` unless profiling shows the creator dispatch is hot. For 99% of factories the construction inside the creator dwarfs the dispatch cost.

<br><br>

---
---

# PART 5: WHEN TO USE WHICH

---
---

<br>

<h2 style="color: #2980B9;">📘 16.11 Decision Table</h2>

| Situation | Use |
|-----------|-----|
| Closed set of types, all known at compile time, won't grow | **Simple Factory** (switch on enum/string) |
| Framework workflow with a "create the product" hook for subclasses | **Factory Method** |
| Multiple **families** of related products, must keep them consistent | **Abstract Factory** |
| Plugin system / open extension / runtime-loaded types | **Registration-Based Factory** |
| Construction needs many parameters, optional fields, validation | **Builder** (Day 17), not factory |
| Single global instance | **Singleton** (Day 15) — but also reach for DI first |

<br>

#### Common combinations in the wild

- **Registration-based factory + Singleton registry** — what most plugin systems are
- **Abstract Factory + Factory Method** — each method on the abstract factory is a factory method
- **Factory + Builder** — factory picks the type, builder configures it (`PizzaFactory::create("margherita").withExtraCheese().build()`)

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 16.12 Factory Anti-Patterns</h2>

#### Anti-pattern 1: A factory for everything

If `Foo` is created in exactly one place with one set of arguments, a `FooFactory` is just an indirection layer. **Use a constructor.**

#### Anti-pattern 2: Factory returning concrete types

```cpp
// Defeats the purpose
std::unique_ptr<Circle> create(...);
```

The whole point is to return the abstract base. If the caller needs the concrete type, they don't need a factory.

#### Anti-pattern 3: Factory that takes the type as a generic parameter

```cpp
template<typename T>
std::unique_ptr<T> create();   // why not just `std::make_unique<T>()`?
```

This is wrapping `make_unique`. Add complexity only when there's polymorphism or registration involved.

#### Anti-pattern 4: Hidden global state via the factory

A registration-based factory **is** global state. If the tests need different registrations than production, you'll need a way to reset or scope the registry. Plan for that on day one.

<br><br>

---
---

# PART 6: BUILD EXERCISE

---
---

<br>

<h2 style="color: #2980B9;">📘 16.13 Exercise: Shape Factory with String Registration</h2>

Build a registration-based shape factory. The driver creates shapes by name without knowing the concrete classes.

<br>

#### Skeleton

```cpp
// shape_factory.h
#pragma once
#include <functional>
#include <memory>
#include <string>
#include <unordered_map>
#include <vector>

class Shape {
public:
    virtual ~Shape() = default;
    virtual void draw() const = 0;
    virtual double area() const = 0;
    virtual std::string name() const = 0;
};

class ShapeFactory {
public:
    using Creator = std::function<std::unique_ptr<Shape>()>;

    static bool registerShape(const std::string& key, Creator creator);
    static std::unique_ptr<Shape> create(const std::string& key);
    static std::vector<std::string> registered();

private:
    static std::unordered_map<std::string, Creator>& registry();
};
```

```cpp
// shape_factory.cpp
#include "shape_factory.h"

std::unordered_map<std::string, ShapeFactory::Creator>& ShapeFactory::registry() {
    static std::unordered_map<std::string, Creator> instance;
    return instance;
}

bool ShapeFactory::registerShape(const std::string& key, Creator creator) {
    return registry().emplace(key, std::move(creator)).second;
}

std::unique_ptr<Shape> ShapeFactory::create(const std::string& key) {
    auto it = registry().find(key);
    return (it == registry().end()) ? nullptr : (it->second)();
}

std::vector<std::string> ShapeFactory::registered() {
    std::vector<std::string> names;
    names.reserve(registry().size());
    for (const auto& [k, _] : registry()) names.push_back(k);
    return names;
}
```

```cpp
// shapes.cpp — concrete shapes self-register
#include "shape_factory.h"
#include <cmath>
#include <iostream>

namespace {

class Circle : public Shape {
public:
    void draw() const override   { std::cout << "drawing circle\n"; }
    double area() const override { return 3.14159 * 1.0 * 1.0; }
    std::string name() const override { return "circle"; }
};

class Square : public Shape {
public:
    void draw() const override   { std::cout << "drawing square\n"; }
    double area() const override { return 1.0 * 1.0; }
    std::string name() const override { return "square"; }
};

class Triangle : public Shape {
public:
    void draw() const override   { std::cout << "drawing triangle\n"; }
    double area() const override { return 0.5 * 1.0 * 1.0; }
    std::string name() const override { return "triangle"; }
};

#define REGISTER_SHAPE(name, ClassType)                                  \
    const bool ClassType##_registered =                                   \
        ShapeFactory::registerShape(name,                                 \
            []() -> std::unique_ptr<Shape> {                              \
                return std::make_unique<ClassType>();                     \
            })

REGISTER_SHAPE("circle",   Circle);
REGISTER_SHAPE("square",   Square);
REGISTER_SHAPE("triangle", Triangle);

} // anonymous namespace
```

<br>

#### Test driver

```cpp
// main.cpp
#include "shape_factory.h"
#include <cassert>
#include <iostream>

int main() {
    std::cout << "=== Registered shapes ===\n";
    for (const auto& n : ShapeFactory::registered()) {
        std::cout << "  - " << n << "\n";
    }
    std::cout << "\n";

    // Create by name
    std::cout << "=== Creating from strings ===\n";
    for (const auto& kind : {"circle", "square", "triangle"}) {
        auto s = ShapeFactory::create(kind);
        assert(s != nullptr);
        s->draw();
        std::cout << "  area = " << s->area() << "\n";
    }
    std::cout << "\n";

    // Unknown shape returns nullptr
    std::cout << "=== Unknown shape ===\n";
    auto nope = ShapeFactory::create("hexagon");
    std::cout << "  hexagon -> " << (nope ? "found" : "nullptr") << "\n";
    std::cout << "\n";

    // Polymorphic use — caller never names a concrete type
    std::cout << "=== Polymorphic loop ===\n";
    std::vector<std::unique_ptr<Shape>> garden;
    for (const auto& k : {"circle", "square", "triangle", "circle"}) {
        garden.push_back(ShapeFactory::create(k));
    }
    double total = 0;
    for (const auto& s : garden) total += s->area();
    std::cout << "  total area = " << total << "\n";

    return 0;
}
```

<br>

#### Compile and run

```bash
g++ -std=c++17 -Wall -Wextra -Wpedantic -fsanitize=address \
    -o day16 main.cpp shape_factory.cpp shapes.cpp && ./day16
```

<br>

#### Expected output pattern

```
=== Registered shapes ===
  - circle
  - square
  - triangle

=== Creating from strings ===
drawing circle
  area = 3.14159
drawing square
  area = 1
drawing triangle
  area = 0.5

=== Unknown shape ===
  hexagon -> nullptr

=== Polymorphic loop ===
  total area = 7.78318
```

<br>

#### Bonus Challenges

1. **Add a Hexagon plugin** — create `hexagon.cpp` with a `Hexagon` class and `REGISTER_SHAPE("hexagon", Hexagon);`. Verify that `main.cpp` and `shape_factory.cpp` need **no changes**.

2. **Constructor parameters** — change the creator signature to `std::function<std::unique_ptr<Shape>(double size)>` so the factory can pass a size argument: `ShapeFactory::create("circle", 5.0)`.

3. **Abstract Factory variant** — build a `ThemeFactory` with `createButton()` and `createScrollbar()` methods. Implement `DarkThemeFactory` and `LightThemeFactory`. Show that mixing produces compile-time-clean but logically inconsistent UIs.

4. **Static-init-order proof** — create two shapes whose registrars are in different `.cpp` files and a third TU that references the registry from another global variable's constructor. Demonstrate that the Meyer's-singleton registry survives any init order.

5. **Linker stripping demo** — compile shapes into a **static library** (`ar rcs libshapes.a shapes.o`) and link the main program **without** `--whole-archive`. Observe that `create("circle")` returns `nullptr` because `Circle` was stripped. Then add `-Wl,--whole-archive` and watch it work.

<br><br>

---
---

# PART 7: DEEP DIVE — COMMON QUESTIONS

---
---

<br>

<h2 style="color: #2980B9;">📘 16.14 Q&A</h2>

<br>

#### Q1: "Is a factory just a fancy way to call `new`?"

For a closed set of types, yes — it's an indirection. The pattern earns its keep when:
- Callers must not know concrete types (decoupling)
- New types should plug in without modifying existing code (open/closed)
- Construction is non-trivial (needs config, validation, dependency wiring)
- Ownership semantics need to be enforced (returning `unique_ptr`)

If none of those apply, just call the constructor.

<br>

#### Q2: "How is Factory different from Builder?"

| | Factory | Builder |
|---|--------|---------|
| **Goal** | Pick **which** type to create | Configure **how** an object is built |
| **Steps** | One call → finished object | Multiple calls → call `build()` |
| **Output** | One of several types | One target type, many variants |
| **Use when** | Type selection by key/condition | Many optional parameters / step-by-step setup |

Often combined: `factory.create("pizza").withExtraCheese().build()`.

<br>

#### Q3: "Why not pass an enum instead of a string for the key?"

Enums are great when types are closed:
```cpp
enum class ShapeKind { Circle, Square, Triangle };
auto s = ShapeFactory::create(ShapeKind::Circle);
```

Strings are required when:
- Keys come from configuration files / network / user input
- Types are added dynamically (plugins) — you can't extend an enum without recompiling

Use enums for type-safety in closed domains; use strings (or `std::variant<EnumA, EnumB, std::string>`) for open extension.

<br>

#### Q4: "What if I need the factory to be testable?"

Two approaches:

**1. Inject the factory as a dependency** (preferred):
```cpp
class OrderProcessor {
    ShapeFactory& m_factory;
public:
    OrderProcessor(ShapeFactory& f) : m_factory(f) {}
};
// In tests, pass a MockShapeFactory that returns canned shapes.
```

**2. Reset the registry between tests**:
```cpp
ShapeFactory::clear();
ShapeFactory::registerShape("circle", []{ return std::make_unique<MockCircle>(); });
```
Add a `clear()` method that empties `registry()`. Be aware: this fights against self-registration via static initializers, since they run only once.

<br>

#### Q5: "Can the factory create objects with constructor arguments?"

Yes — just widen the creator signature:

```cpp
using Creator = std::function<std::unique_ptr<Shape>(double size)>;

static std::unique_ptr<Shape> create(const std::string& name, double size) {
    auto it = registry().find(name);
    return (it == registry().end()) ? nullptr : (it->second)(size);
}
```

For varying argument types per shape, you need:
- `std::any` / `std::variant` parameters (runtime), or
- A per-type config struct passed through a `void*` (avoid), or
- A Builder per type that the factory creates first

<br>

#### Q6: "Why is the registry inside a function instead of a static member?"

Static member objects across translation units have **undefined initialization order**. If a concrete class's registrar runs before the registry is constructed, it inserts into invalid memory — undefined behavior.

Wrapping the registry in a function with a local `static` (Meyer's singleton) defers construction until first call. The first registrar to run triggers construction; subsequent ones see the constructed map. This is the same fix as Day 15's static-initialization-order fiasco.

<br>

#### Q7: "Doesn't `std::function` have overhead? Should I worry?"

Each `std::function` instance may type-erase its callable on the heap (lambdas with captures > small-buffer size) and adds a virtual-like indirect call. For a factory:
- The creator is invoked **once per object created**
- Object construction itself usually dominates the dispatch cost
- The map lookup (`unordered_map::find`) is far more expensive than the call

Don't optimize the creator dispatch unless profiling proves it matters. If it does, switch to a function pointer or a templated registry keyed by `std::type_index`.

<br>

#### Q8: "What about thread safety?"

Two distinct concerns:

**Registration time** — typically static initialization, single-threaded by spec for most platforms. No locking needed.

**Runtime lookup/create** — if you `registerShape` after `main()` starts while other threads call `create`, you need synchronization. Add a `std::shared_mutex`:

```cpp
static std::shared_mutex& mtx() { static std::shared_mutex m; return m; }

static std::unique_ptr<Shape> create(const std::string& name) {
    std::shared_lock lock(mtx());                          // multiple readers
    auto it = registry().find(name);
    return (it == registry().end()) ? nullptr : (it->second)();
}

static bool registerShape(const std::string& name, Creator c) {
    std::unique_lock lock(mtx());                          // exclusive writer
    return registry().emplace(name, std::move(c)).second;
}
```

For static-initialization-only registration (no dynamic plugin loading), locks are unnecessary.

<br><br>

---

## Reflection Questions

1. What problem do factories solve that constructors don't?
2. Why does a registration-based factory uphold the Open/Closed Principle while a switch-based factory violates it?
3. When would you choose Factory Method over Abstract Factory?
4. Why must the factory's registry live inside a static-getter function, not as a static member variable?
5. Why should a factory return `std::unique_ptr<Base>` rather than a raw pointer or a concrete derived type?
6. What linker pitfall can silently disable self-registration, and how do you fix it?

---

## Interview Questions

1. "Implement a factory that creates objects by string name and supports adding new types from plugins without modifying the factory."
2. "What's the difference between Factory Method and Abstract Factory? Give a real-world example of each."
3. "How does a registration-based factory uphold the Open/Closed Principle?"
4. "Why is `std::function` typically used as the creator type? When would you pick a function pointer instead?"
5. "Walk me through the static-initialization-order fiasco in the context of a self-registering factory. How do you avoid it?"
6. "How would you make a factory thread-safe? Distinguish registration-time and lookup-time concurrency."
7. "How do you make a factory testable? Show how you'd inject a mock factory."
8. "If a concrete class is in a static library, why might its registration silently not happen? How do you ensure it does?"
9. "How do you pass constructor arguments through a factory? What are the trade-offs of each approach?"
10. "When is a factory the wrong tool? Compare with Builder, Singleton, and direct construction."

---

**Next**: Day 17 — Builder Pattern →
