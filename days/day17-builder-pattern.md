# Day 17: Builder Pattern

[← Back to Study Plan](../lld-study-plan.md) | [← Day 16](day16-factory-abstract-factory.md)

> **Time**: ~1.5-2 hours
> **Goal**: Understand the Builder pattern — when constructors collapse under too many parameters, how step-by-step construction with a **fluent API** untangles them, and how to produce **immutable result objects** safely. Learn the difference between the classic GoF Builder (Director + Builder + Product) and the modern fluent inner-builder, and how to enforce required fields at compile time with the type-state idiom. Build a `QueryBuilder` with `.select()`, `.from()`, `.where()`, `.build()`.

---
---

# PART 1: WHAT IS THE BUILDER PATTERN AND WHY DOES IT EXIST?

---
---

<br>

<h2 style="color: #2980B9;">📘 17.1 The Core Idea</h2>

A builder constructs a complex object **incrementally**, one step at a time, and yields a finished result at the end via an explicit `build()` call.

```
Constructor:                                  Builder:
  Pizza p("margherita",                         Pizza p = PizzaBuilder()
          true,   // extra cheese                  .name("margherita")
          false,  // anchovies                     .extraCheese(true)
          "thin", // crust                         .crust("thin")
          12,     // diameter                      .diameter(12)
          {"basil","oregano"});                    .toppings({"basil","oregano"})
                                                  .build();
```

The builder fixes three real problems with constructors:
- **Too many parameters** — humans can't read `Pizza("margherita", true, false, "thin", 12, ...)` at the call site
- **Lots of optional fields** — overloads explode (telescoping constructors)
- **Step-by-step construction** — when later steps depend on earlier ones, or pieces arrive over time

<br>

#### The job description

| What a builder is for | What a builder is *not* for |
|----------------------|------------------------------|
| Configuring **one** target type with many fields | Choosing **which** type to create (that's a Factory — Day 16) |
| Replacing painful overload sets | Replacing 2- or 3-parameter constructors |
| Producing an immutable final object | Holding mutable state for the lifetime of the program |
| Validating consistency in `build()` | Validating each setter individually (usually) |

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 17.2 The Symptoms That Demand a Builder</h2>

<br>

#### Symptom 1: Telescoping constructors

```cpp
class HttpRequest {
public:
    HttpRequest(std::string url);
    HttpRequest(std::string url, std::string method);
    HttpRequest(std::string url, std::string method, Headers h);
    HttpRequest(std::string url, std::string method, Headers h, std::string body);
    HttpRequest(std::string url, std::string method, Headers h, std::string body, int timeout);
    HttpRequest(std::string url, std::string method, Headers h, std::string body, int timeout, bool followRedirects);
    // ... and so on
};
```

Five overloads in, you cannot remember which arguments are required, what's optional, or what order they go in. Every new field doubles the misery.

<br>

#### Symptom 2: The "all bool" call site

```cpp
HttpRequest req("https://api.example.com",
                "POST",
                headers,
                body,
                30,
                true,    // followRedirects? verifySSL? compress? — who knows
                false,
                true);
```

Positional arguments lose their names. `true, false, true` is unreadable and easy to mis-order.

<br>

#### Symptom 3: Optional fields with sensible defaults

If 12 of 15 fields have defaults and most callers only set 2-3, a constructor must either:
- Provide a giant default-arg list (still positional)
- Provide many overloads (telescoping)
- Take a config struct (one step toward a builder anyway)

<br>

#### Symptom 4: Construction that needs validation across fields

```cpp
// Invalid: SSL enabled but no certificate path
// Invalid: timeout < 0
// Invalid: method = "GET" but body is non-empty
```

A constructor must throw or do all-up-front validation. A builder defers validation to `build()`, where all fields are visible at once.

<br>

When two or more of these symptoms apply, reach for a builder.

<br><br>

---
---

# PART 2: BUILDER VARIANTS

---
---

<br>

<h2 style="color: #2980B9;">📘 17.3 Variant 1: Classic GoF Builder (Director + Builder + Product)</h2>

The original Gang-of-Four formulation separates **what** to build (Director) from **how** to build it (Builder).

```cpp
// Product
class House {
public:
    void setWalls(const std::string& w)  { m_walls = w; }
    void setRoof(const std::string& r)   { m_roof  = r; }
    void setDoors(int n)                 { m_doors = n; }
    void describe() const {
        std::cout << m_walls << " walls, " << m_roof << " roof, " << m_doors << " doors\n";
    }
private:
    std::string m_walls, m_roof;
    int m_doors = 0;
};

// Abstract builder
class HouseBuilder {
public:
    virtual ~HouseBuilder() = default;
    virtual void buildWalls() = 0;
    virtual void buildRoof()  = 0;
    virtual void buildDoors() = 0;
    virtual House getResult() = 0;
};

// Concrete builder — wood
class WoodHouseBuilder : public HouseBuilder {
    House m_house;
public:
    void buildWalls() override { m_house.setWalls("wooden"); }
    void buildRoof()  override { m_house.setRoof("shingled"); }
    void buildDoors() override { m_house.setDoors(2); }
    House getResult() override { return std::move(m_house); }
};

// Concrete builder — stone
class StoneHouseBuilder : public HouseBuilder {
    House m_house;
public:
    void buildWalls() override { m_house.setWalls("stone"); }
    void buildRoof()  override { m_house.setRoof("slate"); }
    void buildDoors() override { m_house.setDoors(1); }
    House getResult() override { return std::move(m_house); }
};

// Director — owns the recipe (sequence of steps)
class HouseDirector {
public:
    House construct(HouseBuilder& builder) {
        builder.buildWalls();
        builder.buildRoof();
        builder.buildDoors();
        return builder.getResult();
    }
};

// Usage:
HouseDirector director;
WoodHouseBuilder wood;
House woodHouse = director.construct(wood);

StoneHouseBuilder stone;
House stoneHouse = director.construct(stone);
```

<br>

#### When this shape pays off

The Director/Builder split is overkill in 90% of cases. It earns its keep when:
- The **recipe** (order of steps) is the asset, and you want to reuse it across different concrete builders
- Multiple concrete builders produce **structurally similar but materially different** products (e.g., a parser building an AST vs a token stream from the same grammar)

For most day-to-day code — configuring one type with many fields — the modern fluent builder (next) is shorter and clearer.

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 17.4 Variant 2: Modern Fluent Builder</h2>

Method chaining returns `*this` from each setter so calls compose into a sentence. Most C++ code today uses this form.

```cpp
class HttpRequest;  // forward

class HttpRequestBuilder {
public:
    HttpRequestBuilder& url(std::string u)            { m_url = std::move(u);    return *this; }
    HttpRequestBuilder& method(std::string m)         { m_method = std::move(m); return *this; }
    HttpRequestBuilder& header(std::string k, std::string v) {
        m_headers.emplace(std::move(k), std::move(v));
        return *this;
    }
    HttpRequestBuilder& body(std::string b)           { m_body = std::move(b); return *this; }
    HttpRequestBuilder& timeout(int seconds)          { m_timeout = seconds;   return *this; }
    HttpRequestBuilder& followRedirects(bool yes)     { m_redirects = yes;     return *this; }

    HttpRequest build() const;   // defined after HttpRequest

private:
    friend class HttpRequest;
    std::string m_url;
    std::string m_method = "GET";
    std::map<std::string, std::string> m_headers;
    std::string m_body;
    int m_timeout = 30;
    bool m_redirects = true;
};

class HttpRequest {
public:
    // Constructor takes the builder by const reference
    explicit HttpRequest(const HttpRequestBuilder& b)
        : m_url(b.m_url), m_method(b.m_method),
          m_headers(b.m_headers), m_body(b.m_body),
          m_timeout(b.m_timeout), m_redirects(b.m_redirects) {
        if (m_url.empty()) throw std::invalid_argument("url required");
        if (m_timeout < 0) throw std::invalid_argument("timeout < 0");
    }
    // ... accessors ...
private:
    std::string m_url, m_method, m_body;
    std::map<std::string, std::string> m_headers;
    int m_timeout;
    bool m_redirects;
};

HttpRequest HttpRequestBuilder::build() const { return HttpRequest(*this); }

// Usage — reads like prose:
auto req = HttpRequestBuilder()
    .url("https://api.example.com/v1/users")
    .method("POST")
    .header("Content-Type", "application/json")
    .header("Authorization", "Bearer abc123")
    .body(R"({"name":"alice"})")
    .timeout(15)
    .build();
```

<br>

#### Why return `*this` (and not `*this&` or a fresh builder)?

Three options exist; the trade-offs differ:

| Return type | Behavior | Trade-off |
|-------------|----------|-----------|
| `Builder&` | Mutates the builder, returns reference | Cheapest. Builder is reusable but mutable. **Most common choice.** |
| `Builder` (by value) | Returns a modified **copy** | Each step is independent. Allows branching/forking. Higher cost on heavy fields. |
| `const Builder&` (immutable builder, returns new) | Each setter constructs a new builder with one field changed | Pure functional style. Compiles to copies; rarely worth the elegance for typical builders. |

The typical answer is `Builder&`. Use value-returning builders if callers actually fork: `auto base = b.url(...); auto a = base.method("GET"); auto b = base.method("POST");`.

<br>

#### A subtlety: rvalue chains

If callers chain on a temporary (`HttpRequestBuilder().url(...).method(...).build()`), every setter is invoked on an rvalue. Your `Builder&` overload still binds (the temporary is bound through the implicit `this` pointer, which is fine). For zero-friction temporaries, you can ref-qualify and provide an rvalue overload that returns a moved builder, but this is optimization most code doesn't need.

```cpp
// Optional rvalue overload — for power users
HttpRequestBuilder&& url(std::string u) && {
    m_url = std::move(u);
    return std::move(*this);
}
```

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 17.5 Variant 3: Inner Builder ("Static Builder")</h2>

A common idiom — make the builder a **nested type** of the product, with the product's constructor private and the builder declared a friend. The only path to construct the product is `Product::Builder().build()`.

```cpp
class Pizza {
public:
    class Builder;   // forward declaration

    void describe() const {
        std::cout << m_name << ": " << m_diameter << "in";
        if (m_extraCheese) std::cout << ", extra cheese";
        for (const auto& t : m_toppings) std::cout << ", " << t;
        std::cout << "\n";
    }

private:
    Pizza() = default;          // can only be made by the builder
    friend class Builder;

    std::string m_name;
    int m_diameter = 12;
    bool m_extraCheese = false;
    std::vector<std::string> m_toppings;
};

class Pizza::Builder {
public:
    Builder& name(std::string n)              { m_p.m_name = std::move(n); return *this; }
    Builder& diameter(int inches)             { m_p.m_diameter = inches;   return *this; }
    Builder& extraCheese(bool b = true)       { m_p.m_extraCheese = b;     return *this; }
    Builder& topping(std::string t)           { m_p.m_toppings.push_back(std::move(t)); return *this; }

    Pizza build() {
        if (m_p.m_name.empty())   throw std::invalid_argument("name required");
        if (m_p.m_diameter <= 0)  throw std::invalid_argument("diameter must be > 0");
        return std::move(m_p);
    }
private:
    Pizza m_p;
};

// Usage:
auto pizza = Pizza::Builder()
    .name("margherita")
    .diameter(14)
    .extraCheese()
    .topping("basil")
    .topping("tomato")
    .build();
```

<br>

#### Why this shape is popular

- The product type is **immutable from the outside** (no setters)
- Callers can't accidentally construct half-built products — the only constructor is private
- The builder is paired with the product (it's `Pizza::Builder`, not `PizzaBuilder` floating around)
- `build()` is the single validation point

This is the Java/Kotlin "data class with builder" idiom translated to C++, and it's the right default for most "config-heavy product" cases.

<br><br>

---
---

# PART 3: FLUENT API DESIGN

---
---

<br>

<h2 style="color: #2980B9;">📘 17.6 Method Chaining Mechanics</h2>

The chain `b.x().y().z()` parses as `(((b.x()).y()).z())`. For this to compile, `x()` must return something that `.y()` can be called on — usually `Builder&`.

<br>

#### Chaining with overloaded setters

Multiple ways to express the same option are fine, as long as each returns the builder:

```cpp
// Both work, both compose
.header("Content-Type", "application/json")          // single header
.headers({{"X-Foo", "bar"}, {"X-Baz", "qux"}})       // bulk
```

<br>

#### Chaining and `std::move`

If a setter takes its argument by value (cheap to overload, allows perfect forwarding), `std::move` it into the member:

```cpp
Builder& url(std::string u) {
    m_url = std::move(u);  // sink — move into storage
    return *this;
}
```

This handles both lvalue (copy) and rvalue (move) cases optimally. Don't write three overloads; the by-value sink is the C++17 idiom.

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 17.7 Immutable Result Objects</h2>

The product should typically be **immutable after `build()`**. This means:
- No public setters
- All members are accessed through `const` getters
- Modifications produce **new** instances (via a fresh builder)

```cpp
class HttpRequest {
public:
    const std::string& url() const         { return m_url; }
    const std::string& method() const      { return m_method; }
    int timeout() const                    { return m_timeout; }
    // No setUrl(), setMethod(), etc.
private:
    /* members ... */
};
```

<br>

#### Why immutable?

- **Thread-safe by construction** — multiple threads can read without locks
- **No half-built objects in flight** — once you have an `HttpRequest`, every field is set and validated
- **Easier to reason about** — no "did this get mutated since I read it?" questions

For products that genuinely need to mutate, the builder doesn't replace setters — but consider whether the mutation can be a fresh build instead.

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 17.8 Required vs Optional Fields — The Type-State Idiom</h2>

A classic builder bug: forgetting a required field, getting a runtime exception from `build()`. We can move that check to **compile time** with the **type-state pattern** — the builder's type changes as you call setters, and `build()` only exists when the required fields have been provided.

```cpp
// Three states: nothing set → url set → url+method set → can build
template<bool HasUrl, bool HasMethod>
class HttpRequestBuilderT {
    std::string m_url, m_method, m_body;
    int m_timeout = 30;
public:
    HttpRequestBuilderT() = default;

    // url sets HasUrl = true; preserves HasMethod
    HttpRequestBuilderT<true, HasMethod> url(std::string u) && {
        return {std::move(u), std::move(m_method), std::move(m_body), m_timeout};
    }

    HttpRequestBuilderT<HasUrl, true> method(std::string m) && {
        return {std::move(m_url), std::move(m), std::move(m_body), m_timeout};
    }

    HttpRequestBuilderT&& body(std::string b) &&    { m_body = std::move(b); return std::move(*this); }
    HttpRequestBuilderT&& timeout(int t) &&         { m_timeout = t;         return std::move(*this); }

    // build() only exists when both required fields are set
    template<bool U = HasUrl, bool M = HasMethod,
             typename = std::enable_if_t<U && M>>
    HttpRequest build() && { /* construct */ }

private:
    HttpRequestBuilderT(std::string u, std::string m, std::string b, int t)
        : m_url(std::move(u)), m_method(std::move(m)), m_body(std::move(b)), m_timeout(t) {}

    template<bool, bool> friend class HttpRequestBuilderT;
};

using HttpRequestBuilder = HttpRequestBuilderT<false, false>;

// Usage — the compiler enforces required fields:
auto good = HttpRequestBuilder().url("https://x").method("GET").build();   // OK

auto bad  = HttpRequestBuilder().url("https://x").build();                  // COMPILE ERROR
//   ^ build() not callable: HasMethod = false
```

<br>

This is excellent for safety-critical APIs but comes at a cost: more template machinery, harder error messages, slower compile times. Use it when the cost of forgetting a required field is high (e.g., a security-critical config). For most code, runtime validation in `build()` is fine.

<br><br>

---
---

# PART 4: BUILDER VS OTHER PATTERNS

---
---

<br>

<h2 style="color: #2980B9;">📘 17.9 Builder vs Factory vs Constructor — When to Pick Each</h2>

| Situation | Use |
|-----------|-----|
| 2-3 required parameters, no optional fields | **Constructor** |
| Many optional fields, sensible defaults | **Builder** |
| Pick which concrete type to create | **Factory** (Day 16) |
| Pick a type **and** configure it | **Factory + Builder** (factory returns a builder or is chained from one) |
| Configuration sourced from a struct/JSON | **Aggregate / designated initializers** (C++20) |
| Object built incrementally as data arrives over time | **Builder** with explicit lifetime |

<br>

#### Builder vs aggregate initialization (C++20 designated initializers)

C++20 lets you do this:

```cpp
struct HttpConfig {
    std::string url;
    std::string method = "GET";
    int timeout = 30;
    bool followRedirects = true;
};

HttpConfig cfg{
    .url = "https://api.example.com",
    .method = "POST",
    .timeout = 15,
};
```

When does this beat a builder?
- ✅ Pure data with no validation logic
- ✅ Field order doesn't matter (designators are named)
- ✅ Compile-time checking of field names (typo = error)

When does a builder still win?
- ❌ Construction has computed/derived fields
- ❌ Validation must happen across multiple fields
- ❌ Some setters take multiple args (like `header(key, value)`)
- ❌ The product needs to be immutable but the builder needs intermediate state

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 17.10 Builder Anti-Patterns</h2>

#### Anti-pattern 1: A builder for two-field types

```cpp
auto p = PointBuilder().x(3).y(4).build();   // really? just write Point{3, 4}
```

If the constructor reads fine, the builder is pure noise.

#### Anti-pattern 2: Skipping `build()` (auto-finalization)

Some libraries make the builder *implicitly convertible* to the product. This eliminates the explicit finalization point and lets callers skip validation by accident:

```cpp
HttpRequest req = HttpRequestBuilder().url("...");   // converted silently — bug magnet
```

Keep `build()` explicit. It's a forcing function for thinking "is everything set?".

#### Anti-pattern 3: Builder with public mutable getters

If callers can read partial state mid-build and branch on it, the builder has become an event-sourcing engine. Either keep state private until `build()`, or switch to a fully observable pattern.

#### Anti-pattern 4: Validation-only-in-`build()` for all errors

Some errors are **immediate** (negative timeout, malformed URL). Validating those in the setter gives a stack trace pointing at the broken call site, not at `build()` 30 lines down. Validate cross-field invariants in `build()`; validate single-field invariants in the setter.

<br><br>

---
---

# PART 5: VALIDATION & ERROR HANDLING

---
---

<br>

<h2 style="color: #2980B9;">📘 17.11 Where to Validate</h2>

| Kind of error | Validate where? | Why |
|---------------|-----------------|-----|
| Single-field domain (e.g., `timeout < 0`) | In the setter | Stack trace points at the offender |
| Cross-field consistency (e.g., `method=="GET"` but `body!=""`) | In `build()` | All fields are visible together |
| Required field missing | In `build()` (or compile time via type-state) | Setter doesn't know "missing" until the end |
| External I/O (e.g., file exists) | **Not in the builder** | Builder shouldn't have side effects |

<br>

#### Throwing vs result type

```cpp
// Option A — throw on invalid build
HttpRequest build() const {
    if (m_url.empty()) throw std::invalid_argument("url required");
    return HttpRequest(*this);
}

// Option B — return std::optional / expected
std::optional<HttpRequest> tryBuild() const {
    if (m_url.empty()) return std::nullopt;
    return HttpRequest(*this);
}
```

Throwing is the C++ default and matches `std::invalid_argument`'s purpose. Use the `optional`/`expected` form when invalid configurations are expected during normal flow (e.g., parsing user input).

<br>

#### Don't validate side effects in `build()`

```cpp
HttpRequest build() const {
    // BAD — opens a socket inside build()
    auto sock = openSocket(m_url);
    if (!sock) throw ConnectionFailed{};
    return HttpRequest(*this, std::move(sock));
}
```

The builder configures; an external operation connects. Keep them separate or you cripple test/replay scenarios.

<br><br>

---
---

# PART 6: BUILD EXERCISE

---
---

<br>

<h2 style="color: #2980B9;">📘 17.12 Exercise: <code>QueryBuilder</code> — Fluent SQL Builder</h2>

Build a SQL `SELECT` builder with `.select()`, `.from()`, `.where()`, `.orderBy()`, `.limit()`, and `.build()` returning a query string.

<br>

#### Skeleton

```cpp
// query_builder.h
#pragma once
#include <optional>
#include <sstream>
#include <stdexcept>
#include <string>
#include <vector>

class QueryBuilder {
public:
    QueryBuilder& select(std::vector<std::string> cols) {
        m_columns = std::move(cols);
        return *this;
    }

    QueryBuilder& from(std::string table) {
        if (table.empty()) throw std::invalid_argument("from(): empty table");
        m_table = std::move(table);
        return *this;
    }

    // Each call appends an AND-combined predicate
    QueryBuilder& where(std::string predicate) {
        m_conditions.push_back(std::move(predicate));
        return *this;
    }

    QueryBuilder& orderBy(std::string column, bool ascending = true) {
        m_orderColumn = std::move(column);
        m_orderAscending = ascending;
        return *this;
    }

    QueryBuilder& limit(int n) {
        if (n < 0) throw std::invalid_argument("limit < 0");
        m_limit = n;
        return *this;
    }

    std::string build() const {
        if (m_table.empty())     throw std::logic_error("build(): from() not set");
        if (m_columns.empty())   throw std::logic_error("build(): select() not set");

        std::ostringstream out;
        out << "SELECT ";
        for (size_t i = 0; i < m_columns.size(); ++i) {
            out << (i ? ", " : "") << m_columns[i];
        }
        out << " FROM " << m_table;

        if (!m_conditions.empty()) {
            out << " WHERE ";
            for (size_t i = 0; i < m_conditions.size(); ++i) {
                out << (i ? " AND " : "") << "(" << m_conditions[i] << ")";
            }
        }

        if (m_orderColumn) {
            out << " ORDER BY " << *m_orderColumn
                << (m_orderAscending ? " ASC" : " DESC");
        }

        if (m_limit) {
            out << " LIMIT " << *m_limit;
        }

        return out.str();
    }

private:
    std::vector<std::string> m_columns;
    std::string m_table;
    std::vector<std::string> m_conditions;
    std::optional<std::string> m_orderColumn;
    bool m_orderAscending = true;
    std::optional<int> m_limit;
};
```

<br>

#### Test driver

```cpp
// main.cpp
#include "query_builder.h"
#include <cassert>
#include <iostream>

void expect(const std::string& actual, const std::string& expected) {
    if (actual != expected) {
        std::cerr << "MISMATCH:\n  got:      " << actual
                  << "\n  expected: " << expected << "\n";
        std::abort();
    }
    std::cout << "OK: " << actual << "\n";
}

int main() {
    // 1. Simple select
    expect(
        QueryBuilder().select({"id", "name"}).from("users").build(),
        "SELECT id, name FROM users"
    );

    // 2. With single where
    expect(
        QueryBuilder()
            .select({"id"})
            .from("users")
            .where("age > 18")
            .build(),
        "SELECT id FROM users WHERE (age > 18)"
    );

    // 3. Multiple wheres are AND-combined
    expect(
        QueryBuilder()
            .select({"id", "email"})
            .from("users")
            .where("age > 18")
            .where("country = 'US'")
            .build(),
        "SELECT id, email FROM users WHERE (age > 18) AND (country = 'US')"
    );

    // 4. Order + limit
    expect(
        QueryBuilder()
            .select({"id", "score"})
            .from("scores")
            .orderBy("score", false)
            .limit(10)
            .build(),
        "SELECT id, score FROM scores ORDER BY score DESC LIMIT 10"
    );

    // 5. Everything together
    expect(
        QueryBuilder()
            .select({"u.id", "u.name", "o.total"})
            .from("users u JOIN orders o ON o.user_id = u.id")
            .where("o.created_at > '2025-01-01'")
            .where("o.total > 100")
            .orderBy("o.total")
            .limit(50)
            .build(),
        "SELECT u.id, u.name, o.total FROM users u JOIN orders o ON o.user_id = u.id "
        "WHERE (o.created_at > '2025-01-01') AND (o.total > 100) "
        "ORDER BY o.total ASC LIMIT 50"
    );

    // 6. Validation errors
    try {
        QueryBuilder().select({"id"}).build();   // no from
        std::abort();
    } catch (const std::logic_error& e) {
        std::cout << "OK threw: " << e.what() << "\n";
    }

    try {
        QueryBuilder().from("users").build();    // no select
        std::abort();
    } catch (const std::logic_error& e) {
        std::cout << "OK threw: " << e.what() << "\n";
    }

    try {
        QueryBuilder().select({"id"}).from("users").limit(-1);
        std::abort();
    } catch (const std::invalid_argument& e) {
        std::cout << "OK threw: " << e.what() << "\n";
    }

    return 0;
}
```

<br>

#### Compile and run

```bash
g++ -std=c++17 -Wall -Wextra -Wpedantic -fsanitize=address \
    -o day17 main.cpp && ./day17
```

<br>

#### Expected output pattern

```
OK: SELECT id, name FROM users
OK: SELECT id FROM users WHERE (age > 18)
OK: SELECT id, email FROM users WHERE (age > 18) AND (country = 'US')
OK: SELECT id, score FROM scores ORDER BY score DESC LIMIT 10
OK: SELECT u.id, u.name, o.total FROM users u JOIN orders o ON o.user_id = u.id WHERE (o.created_at > '2025-01-01') AND (o.total > 100) ORDER BY o.total ASC LIMIT 50
OK threw: build(): from() not set
OK threw: build(): select() not set
OK threw: limit < 0
```

<br>

#### Bonus Challenges

1. **Parameterized predicates** — change `where()` to accept a column, operator, and a typed value (e.g. `where("age", ">", 18)`). Generate placeholder params (`?`) and return `(query, args)` from `build()` to enable safe parameterized queries.

2. **`groupBy` and `having`** — extend the builder. `having` is only valid if `groupBy` is set; enforce this in `build()`.

3. **Type-state required fields** — convert `select` and `from` into compile-time required fields using the type-state idiom from §17.8. Try compiling without one of them — verify the compiler error.

4. **Inner builder form** — refactor so the API is `Query::Builder().select(...)...build()` and `Query` has a private constructor. Make `Query` immutable.

5. **Branching reuse** — change every setter to return `QueryBuilder` by value (a copy). Demonstrate that:
   ```cpp
   auto base = QueryBuilder().select({"id"}).from("users");
   auto adults  = base.where("age >= 18").build();
   auto minors  = base.where("age <  18").build();
   ```
   produces two distinct queries from the same base. Compare allocations vs the by-reference variant.

6. **Subquery support** — let `from()` accept another `QueryBuilder` (a subquery), e.g. `from(QueryBuilder().select({"*"}).from("raw").where("flag=1"))`. Compose recursively.

<br><br>

---
---

# PART 7: DEEP DIVE — COMMON QUESTIONS

---
---

<br>

<h2 style="color: #2980B9;">📘 17.13 Q&A</h2>

<br>

#### Q1: "When is a builder NOT worth the boilerplate?"

If your type has:
- ≤ 3 parameters, all required, all unambiguous → use a constructor
- A small set of params + you're on C++20 → consider designated initializers on a config struct
- One required field and one common optional → use a constructor with a default arg

A builder is justified when callers find the constructor confusing or write helper functions to wrap it. Those workarounds are the symptom; the builder is the cure.

<br>

#### Q2: "Should `build()` return by value, by `unique_ptr`, or move?"

Default to **by value**. The Named Return Value Optimization (NRVO) elides the copy. If the product is non-copyable but movable, that's fine — it'll move. Use `unique_ptr` only when:
- The product is polymorphic (returned through a base pointer)
- The product is large and you want explicit allocation control
- The factory pattern logic is mixed in (you're picking the type)

<br>

#### Q3: "How do I make the builder reusable across multiple `build()` calls?"

By default, a fluent `Builder&` builder mutates state, so calling `build()` twice produces two products from the *same* state — usually fine. If you want **independent** builds with shared base config, return by value:

```cpp
QueryBuilder base = QueryBuilder().select({"id"}).from("users");
auto a = base.where("age > 18").build();   // base unchanged if setters return by value
auto b = base.where("age > 65").build();   // different where
```

If your builder mutates state (`Builder&`), chaining off `base` would mutate it. Pick one model and document it.

<br>

#### Q4: "Builder vs named-arguments libraries (e.g., Boost.Parameter)?"

Builders are explicit and require no metaprogramming. Boost.Parameter and similar give you keyword-argument-like syntax at construction (`Pizza(_name="margherita", _diameter=14)`) but at the cost of heavy templates and impenetrable error messages. In modern C++, designated initializers (C++20) cover most named-argument cases without machinery; reach for a builder when validation or computed fields are involved.

<br>

#### Q5: "Can I use a builder for objects that mutate over their lifetime?"

You can, but be careful: the *builder* models the construction phase; the *product* models the operating phase. If callers want to "reconfigure" the product, the cleanest pattern is to expose `toBuilder()` on the product, modify the returned builder, and `build()` a new product. This keeps the product immutable.

<br>

#### Q6: "How do I express required vs optional fields without templates?"

Without the type-state idiom, runtime validation is the standard:

```cpp
Pizza Pizza::Builder::build() {
    if (m_name.empty())  throw std::invalid_argument("name required");
    if (m_diameter <= 0) throw std::invalid_argument("diameter must be > 0");
    return std::move(m_p);
}
```

Document required fields clearly in the API. If callers frequently forget them, escalate to type-state.

<br>

#### Q7: "Is a builder thread-safe?"

**No** — builders mutate state across calls. Don't share a builder across threads. Once you `build()`, the resulting product can be made immutable and thread-safe by construction. This is the typical concurrency story: build single-threaded, share the immutable product.

<br>

#### Q8: "What's the relationship between Builder and Fluent Interface?"

A **fluent interface** is a UI style — chained method calls that read like prose. A **builder** is a creational pattern with a `build()` finalization step. Most modern builders are fluent, but you can have:
- A non-fluent builder (separate setter calls, then `build()`) — works fine, just verbose
- A fluent interface that's not a builder (e.g., jQuery's `$('#x').addClass('y').show()` — operates on existing state)

They're orthogonal; combining them gives you the modern C++ builder.

<br>

#### Q9: "How do I unit-test a builder?"

Three layers:
1. **Each setter** stores the right value and returns the builder reference (assert `&result == &builder`)
2. **`build()`** rejects invalid configurations (each `throw` path)
3. **`build()`** produces a product whose accessors return the expected values

Property-based testing (varying the field set) is excellent for builders since the combinatorial space is large.

<br><br>

---

## Reflection Questions

1. What problem do builders solve that constructors and factories don't?
2. When does the classic Director/Builder/Product split pay off vs the modern fluent builder?
3. Why is returning `Builder&` from setters the default? When would you return by value?
4. Why should the result of `build()` typically be immutable?
5. What is the type-state idiom, and what does it buy you over runtime validation?
6. Where should single-field validation happen, and where should cross-field validation happen?

---

## Interview Questions

1. "Implement a `QueryBuilder` with `select`, `from`, `where`, `orderBy`, `limit`, and `build`."
2. "What's the difference between Builder and Factory? Can you combine them?"
3. "When would you choose a builder over named/designated initializers?"
4. "Walk me through making a fluent builder thread-safe. (Trick — don't.)"
5. "Show me how to enforce required fields at compile time using the type-state idiom."
6. "If `build()` is called twice on the same fluent builder, what happens? Is that desirable?"
7. "Describe the telescoping-constructor problem and how a builder solves it."
8. "Where would you place validation in a builder, and why?"
9. "How would you implement a builder that returns an `unique_ptr` to a polymorphic product?"
10. "What's the cost of a builder vs a constructor? When does that cost start to matter?"

---

**Next**: Day 18 — Observer Pattern →
