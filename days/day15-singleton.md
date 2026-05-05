# Day 15: Singleton Pattern

[← Back to Study Plan](../lld-study-plan.md) | [← Day 13](day13-object-pool-pattern.md)

> **Time**: ~1.5-2 hours
> **Goal**: Understand the Singleton pattern deeply — why it exists, why it's controversial, and how to implement it correctly in C++. Learn Meyer's singleton, `std::call_once`, and why double-checked locking was broken before C++11. Test thread safety by racing 10 threads to `getInstance()`.

---
---

# PART 1: WHAT IS A SINGLETON AND WHY DOES IT EXIST?

---
---

<br>

<h2 style="color: #2980B9;">📘 15.1 The Core Idea</h2>

A singleton ensures that a class has **exactly one instance** and provides a **global access point** to it.

```
Normal class:
  Logger a;    // instance 1
  Logger b;    // instance 2
  Logger c;    // instance 3
  → three independent loggers with separate state

Singleton class:
  Logger& a = Logger::getInstance();  // THE instance
  Logger& b = Logger::getInstance();  // same instance
  Logger& c = Logger::getInstance();  // same instance
  → &a == &b == &c — all references to the same object
```

<br>

#### When singletons make sense

| Use Case | Why Single Instance |
|----------|-------------------|
| **Logger** | All code should write to the same log output |
| **Configuration** | One set of config values, read at startup |
| **Thread pool** | One pool shared by the entire application |
| **Connection pool** | One pool of DB connections |
| **Hardware interface** | One object representing a physical device |

<br>

#### When singletons are bad

Singletons are often **overused**. They're essentially **global variables with extra steps**. The problems:

| Problem | Explanation |
|---------|-------------|
| **Hidden dependencies** | Functions secretly depend on global state — hard to trace |
| **Testing difficulty** | Can't replace the singleton with a mock in unit tests |
| **Lifetime issues** | Destruction order of static objects is undefined across translation units |
| **Concurrency** | Shared mutable global state = data race magnet |

**Rule of thumb**: If you can pass the object as a parameter, do that instead. Use a singleton only when there's a genuine reason for exactly one instance.

<br><br>

---
---

# PART 2: IMPLEMENTATION VARIANTS

---
---

<br>

<h2 style="color: #2980B9;">📘 15.2 Variant 1: Meyer's Singleton (The Modern C++ Way)</h2>

Named after Scott Meyers. Uses a **local static variable** — the simplest and safest approach since C++11:

```cpp
class Logger {
public:
    static Logger& getInstance() {
        static Logger instance;  // created on first call, destroyed at program exit
        return instance;
    }

    void log(const std::string& message) {
        std::lock_guard<std::mutex> lock(m_mutex);
        std::cout << "[LOG] " << message << "\n";
    }

    // Delete copy and move to prevent duplicates
    Logger(const Logger&) = delete;
    Logger& operator=(const Logger&) = delete;
    Logger(Logger&&) = delete;
    Logger& operator=(Logger&&) = delete;

private:
    Logger() {
        std::cout << "Logger constructed\n";
    }

    ~Logger() {
        std::cout << "Logger destroyed\n";
    }

    std::mutex m_mutex;
};

// Usage:
Logger::getInstance().log("Server started");
```

<br>

#### Why this works — C++11 guarantees

The C++11 standard (§6.7) guarantees:

> If control enters the declaration of a local static variable concurrently, **only one thread performs the initialization**. Other threads block until initialization is complete.

```
Thread A enters getInstance():           Thread B enters getInstance():
  static Logger instance;                  static Logger instance;
       │                                        │
  Starts constructing Logger             Sees "already being initialized"
       │                                        │
  Constructor runs...                    BLOCKS (waits)
       │                                        │
  Construction complete                  Unblocks — returns same instance
       │                                        │
  return instance;                       return instance;
```

The compiler generates a hidden flag + mutex for each local static:

```cpp
// What the compiler roughly generates:
static bool __guard = false;
static char __storage[sizeof(Logger)];

if (!__guard) {
    // Thread-safe initialization (uses internal mutex or atomic)
    lock(__guard_mutex);
    if (!__guard) {
        new (__storage) Logger();  // placement new
        __guard = true;
        atexit([] { reinterpret_cast<Logger*>(__storage)->~Logger(); });
    }
    unlock(__guard_mutex);
}
return *reinterpret_cast<Logger*>(__storage);
```

**This is the recommended approach in modern C++.** Simple, correct, thread-safe.

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 15.3 Variant 2: <code>std::call_once</code> + <code>std::once_flag</code></h2>

An alternative that gives you **explicit control** over when initialization happens:

```cpp
class Database {
public:
    static Database& getInstance() {
        std::call_once(s_init_flag, []() {
            s_instance.reset(new Database());
        });
        return *s_instance;
    }

    void query(const std::string& sql) {
        std::cout << "Executing: " << sql << "\n";
    }

    Database(const Database&) = delete;
    Database& operator=(const Database&) = delete;

private:
    Database() {
        std::cout << "Database: connecting...\n";
        // Simulate expensive initialization
    }

    ~Database() {
        std::cout << "Database: disconnecting...\n";
    }

    static std::once_flag s_init_flag;
    static std::unique_ptr<Database> s_instance;
};

// In .cpp file:
std::once_flag Database::s_init_flag;
std::unique_ptr<Database> Database::s_instance;
```

<br>

#### How `std::call_once` works

```
Thread A calls getInstance():          Thread B calls getInstance():
  call_once(flag, lambda)                call_once(flag, lambda)
       │                                      │
  flag not set → runs lambda             flag being set → BLOCKS
       │                                      │
  new Database()                         waiting...
       │                                      │
  sets flag to "done"                    unblocks, lambda NOT called
       │                                      │
  return *s_instance                     return *s_instance
```

`std::call_once` guarantees:
- The callable runs **exactly once**, even with concurrent callers
- If the callable throws, it's **not** considered called — the next caller gets to try
- After successful call, subsequent calls are a cheap atomic read (no locking)

<br>

#### When to prefer `call_once` over Meyer's

| Scenario | Preference |
|----------|-----------|
| Simple singleton | Meyer's — less code |
| Need to control destruction order | `call_once` + `unique_ptr` — you can `reset()` manually |
| Need to re-initialize after failure | `call_once` — if constructor throws, next call retries |
| Need to pass constructor arguments | `call_once` — lambda captures arguments |

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 15.4 Variant 3: Eager Initialization (No Lazy Loading)</h2>

If the singleton is always needed, initialize it **before `main()`** via a static global:

```cpp
class Config {
public:
    static Config& getInstance() {
        return s_instance;
    }

    std::string get(const std::string& key) const {
        auto it = m_values.find(key);
        return (it != m_values.end()) ? it->second : "";
    }

    Config(const Config&) = delete;
    Config& operator=(const Config&) = delete;

private:
    Config() {
        // Load config at startup
        m_values["port"] = "8080";
        m_values["host"] = "localhost";
        std::cout << "Config loaded\n";
    }

    std::unordered_map<std::string, std::string> m_values;
    static Config s_instance;  // initialized before main()
};

// In .cpp file:
Config Config::s_instance;  // constructor runs during static initialization
```

<br>

#### Pros and cons

| | Eager | Lazy (Meyer's / call_once) |
|---|-------|---------------------------|
| **When constructed** | Before `main()` | On first use |
| **Thread safety** | N/A — single-threaded at that point | Guaranteed by C++11 |
| **Startup cost** | Pays upfront even if never used | Only if actually used |
| **Static init order** | **DANGEROUS** — order across TUs is undefined | No problem — created on demand |

<br>

#### The Static Initialization Order Fiasco

```cpp
// file_a.cpp
Config Config::s_instance;  // constructed first? second? UNDEFINED!

// file_b.cpp
Logger Logger::s_instance;  // if Logger's constructor uses Config → 
                             // Config might not exist yet → UB!
```

This is a real bug — two eagerly-initialized singletons in different `.cpp` files have **undefined construction order**. If one depends on the other, you get UB.

Meyer's singleton avoids this entirely — it's constructed on first use, so the dependency is resolved naturally:

```cpp
Logger::Logger() {
    // Config::getInstance() will construct Config here if needed
    auto port = Config::getInstance().get("port");
}
```

<br><br>

---
---

# PART 3: THE BROKEN APPROACHES

---
---

<br>

<h2 style="color: #2980B9;">📘 15.5 Double-Checked Locking Pattern (DCLP) — Why It Was Broken</h2>

Before C++11, people tried to make singletons thread-safe with **double-checked locking**:

```cpp
// PRE-C++11 — BROKEN!
class Singleton {
    static Singleton* s_instance;
    static std::mutex s_mutex;

public:
    static Singleton* getInstance() {
        if (s_instance == nullptr) {           // First check (no lock)
            std::lock_guard<std::mutex> lock(s_mutex);
            if (s_instance == nullptr) {       // Second check (with lock)
                s_instance = new Singleton();
            }
        }
        return s_instance;
    }
};
```

The idea: avoid locking on every call by checking the pointer first. Only lock if it's null. After locking, check again (another thread might have initialized it while we waited).

**Why it's broken without C++11 memory model:**

```
Thread A:                              Thread B:
  lock mutex                           
  s_instance = new Singleton()         
                                       
  What "new Singleton()" actually does:
    1. Allocate memory                 
    2. Construct Singleton in memory   
    3. Assign address to s_instance    
                                       
  But the compiler/CPU can REORDER     
  steps 2 and 3:                       
    1. Allocate memory                 
    3. Assign address to s_instance    ← pointer is now non-null!
                                         if (s_instance == nullptr) → false
                                         return s_instance; 
                                         → RETURNS POINTER TO UNCONSTRUCTED OBJECT!
    2. Construct Singleton in memory   ← hasn't happened yet for Thread B
```

The CPU and compiler can reorder the write to `s_instance` before the constructor completes. Thread B sees a non-null pointer to an **uninitialized object** — undefined behavior.

<br>

#### Fixed with C++11 atomics

```cpp
class Singleton {
    static std::atomic<Singleton*> s_instance;
    static std::mutex s_mutex;

public:
    static Singleton* getInstance() {
        Singleton* ptr = s_instance.load(std::memory_order_acquire);
        if (!ptr) {
            std::lock_guard<std::mutex> lock(s_mutex);
            ptr = s_instance.load(std::memory_order_relaxed);
            if (!ptr) {
                ptr = new Singleton();
                s_instance.store(ptr, std::memory_order_release);
            }
        }
        return ptr;
    }
};
```

- `memory_order_release` on the store ensures that the constructor's writes are visible before the pointer is published
- `memory_order_acquire` on the load ensures that after reading a non-null pointer, all constructor writes are visible

**But why bother?** Meyer's singleton does all of this for you with one line. DCLP is only worth studying to understand **why** it was broken — it's a classic interview question.

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 15.6 Other Broken Approaches</h2>

<br>

#### Raw global pointer without any protection

```cpp
// BROKEN — data race on s_instance
static Singleton* s_instance = nullptr;

Singleton* getInstance() {
    if (!s_instance) {
        s_instance = new Singleton();  // two threads can both enter here
    }
    return s_instance;
}
// Result: two Singletons created, one leaked, s_instance points to the second
```

<br>

#### Locking on every call

```cpp
// CORRECT but SLOW — mutex acquired on every single call
static Singleton* s_instance = nullptr;
static std::mutex s_mutex;

Singleton* getInstance() {
    std::lock_guard<std::mutex> lock(s_mutex);
    if (!s_instance) {
        s_instance = new Singleton();
    }
    return s_instance;
}
// After initialization, the mutex is pure overhead
// Thousands of calls per second → unnecessary contention
```

This is correct but defeats the purpose — Meyer's singleton has zero overhead after initialization (the compiler's internal guard is just an atomic read).

<br><br>

---
---

# PART 4: DESTRUCTION AND LIFETIME

---
---

<br>

<h2 style="color: #2980B9;">📘 15.7 When Is the Singleton Destroyed?</h2>

| Variant | Destruction |
|---------|-------------|
| **Meyer's** (local static) | Destroyed at program exit, in **reverse order** of construction (like stack unwinding) |
| **`call_once` + `unique_ptr`** | When `unique_ptr` is destroyed (at static destruction time), or manually via `reset()` |
| **Eager** (global static) | At program exit, order across TUs is **undefined** |
| **Raw `new`** | **Never** — leaked (unless you manually `delete`) |

<br>

#### The Dead Reference Problem

```cpp
class Logger {
    static Logger& getInstance() {
        static Logger instance;
        return instance;
    }
};

class Database {
    ~Database() {
        // This runs during program shutdown
        Logger::getInstance().log("Database shutting down");
        // If Logger was already destroyed → UNDEFINED BEHAVIOR!
        // The static Logger is dead — you're accessing destroyed memory
    }
};
```

If `Database` is destroyed **after** `Logger` (both are static), the `Database` destructor uses a dead `Logger`. This is called the **dead reference problem**.

<br>

#### Solutions

**1. Phoenix singleton** — resurrects if accessed after destruction (rare, complex):

```cpp
static Logger& getInstance() {
    static Logger* instance = nullptr;
    static bool destroyed = false;

    if (!instance) {
        if (destroyed) {
            // Resurrect!
            new (instance) Logger();
            destroyed = false;
            std::atexit([] { 
                instance->~Logger();
                destroyed = true;
            });
        } else {
            static Logger local;
            instance = &local;
        }
    }
    return *instance;
}
```

**2. Ensure correct destruction order** — access dependencies in constructors (Meyer's guarantee: constructed first = destroyed last):

```cpp
Logger::Logger() {
    // By calling Config here, we ensure Config is constructed FIRST
    // Therefore Config is destroyed AFTER Logger (reverse order)
    Config::getInstance();
}
```

**3. Intentional leak** — never destroy (acceptable for loggers and similar):

```cpp
static Logger& getInstance() {
    static Logger* instance = new Logger();  // never deleted
    return *instance;
    // Pros: no destruction order issues
    // Cons: sanitizers report a "leak" (suppress with annotations)
}
```

<br><br>

---
---

# PART 5: MAKING SINGLETONS TESTABLE

---
---

<br>

<h2 style="color: #2980B9;">📘 15.8 The Testability Problem</h2>

Singletons are hard to unit test because you can't replace them with mocks:

```cpp
void processOrder(Order& order) {
    // Hidden dependency on global singleton — can't test without a real database
    Database::getInstance().save(order);
    Logger::getInstance().log("Order processed");
}
```

<br>

#### Solution: Dependency Injection + Interface

```cpp
class ILogger {
public:
    virtual ~ILogger() = default;
    virtual void log(const std::string& msg) = 0;
};

class RealLogger : public ILogger {
public:
    void log(const std::string& msg) override {
        std::cout << "[LOG] " << msg << "\n";
    }
};

class MockLogger : public ILogger {
public:
    std::vector<std::string> messages;
    void log(const std::string& msg) override {
        messages.push_back(msg);
    }
};

// Pass the dependency instead of hardcoding a singleton
void processOrder(Order& order, ILogger& logger, IDatabase& db) {
    db.save(order);
    logger.log("Order processed");
}

// Production:
RealLogger logger;
processOrder(order, logger, realDb);

// Test:
MockLogger mock;
processOrder(order, mock, mockDb);
assert(mock.messages.back() == "Order processed");
```

If you **must** use a singleton, at least make it swappable:

```cpp
class Logger {
    static ILogger* s_instance;
public:
    static ILogger& getInstance() { return *s_instance; }
    static void setInstance(ILogger* impl) { s_instance = impl; }  // for testing
};

// In test:
MockLogger mock;
Logger::setInstance(&mock);
```

<br><br>

---
---

# PART 6: BUILD EXERCISE

---
---

<br>

<h2 style="color: #2980B9;">📘 15.9 Exercise: Implement 3 Singleton Variants</h2>

Implement all three variants and test their thread safety.

<br>

#### Variant A: Meyer's Singleton

```cpp
class Logger {
public:
    static Logger& getInstance();
    void log(const std::string& msg);

    Logger(const Logger&) = delete;
    Logger& operator=(const Logger&) = delete;

private:
    Logger();
    ~Logger();
    std::mutex m_mutex;
};
```

<br>

#### Variant B: `std::call_once`

```cpp
class Database {
public:
    static Database& getInstance();
    void query(const std::string& sql);

    Database(const Database&) = delete;
    Database& operator=(const Database&) = delete;

private:
    Database();
    ~Database();
    static std::once_flag s_flag;
    static std::unique_ptr<Database> s_instance;
};
```

<br>

#### Variant C: Eager Initialization

```cpp
class Config {
public:
    static Config& getInstance();
    std::string get(const std::string& key) const;

    Config(const Config&) = delete;
    Config& operator=(const Config&) = delete;

private:
    Config();
    std::unordered_map<std::string, std::string> m_data;
    static Config s_instance;
};
```

<br>

#### Thread Safety Test Driver

```cpp
#include <thread>
#include <vector>
#include <iostream>
#include <cassert>

void test_meyers_thread_safety() {
    std::cout << "=== Meyer's Singleton Thread Safety ===\n";

    std::vector<Logger*> addresses(10);
    std::vector<std::thread> threads;

    for (int i = 0; i < 10; ++i) {
        threads.emplace_back([&addresses, i]() {
            addresses[i] = &Logger::getInstance();
            Logger::getInstance().log("Thread " + std::to_string(i) + " logging");
        });
    }

    for (auto& t : threads) t.join();

    // All threads must get the same instance
    for (int i = 1; i < 10; ++i) {
        assert(addresses[i] == addresses[0]);
    }
    std::cout << "  All 10 threads got the same instance: " << addresses[0] << "\n";
    std::cout << "  PASSED\n\n";
}

void test_call_once_thread_safety() {
    std::cout << "=== call_once Singleton Thread Safety ===\n";

    std::vector<Database*> addresses(10);
    std::vector<std::thread> threads;

    for (int i = 0; i < 10; ++i) {
        threads.emplace_back([&addresses, i]() {
            addresses[i] = &Database::getInstance();
        });
    }

    for (auto& t : threads) t.join();

    for (int i = 1; i < 10; ++i) {
        assert(addresses[i] == addresses[0]);
    }
    std::cout << "  All 10 threads got the same instance: " << addresses[0] << "\n";
    std::cout << "  PASSED\n\n";
}

int main() {
    test_meyers_thread_safety();
    test_call_once_thread_safety();

    // Eager — already exists before main
    std::cout << "=== Eager Singleton ===\n";
    std::cout << "  port = " << Config::getInstance().get("port") << "\n";

    return 0;
}
```

```bash
g++ -std=c++17 -Wall -Wextra -pthread -fsanitize=thread -o day15 day15_singleton.cpp && ./day15
```

<br>

#### Bonus Challenges

1. **CRTP Singleton base**: Write a `Singleton<T>` base class using CRTP, so any class can become a singleton by inheriting from `Singleton<MyClass>`.

2. **Swappable singleton for testing**: Add `setInstance()` / `resetInstance()` to Meyer's singleton to allow replacing with mocks in tests.

3. **Destruction order test**: Create two singletons where A depends on B. Verify construction and destruction order. Then try the reverse — observe the dead reference problem.

4. **Thread-safe DCLP**: Implement double-checked locking with `std::atomic` and correct memory ordering. Compare the generated assembly (on godbolt.org) with Meyer's singleton.

<br><br>

---
---

# PART 7: DEEP DIVE — COMMON QUESTIONS

---
---

<br>

<h2 style="color: #2980B9;">📘 15.10 Q&A</h2>

<br>

#### Q1: "Is Meyer's singleton truly zero-overhead after first call?"

Almost. The compiler generates an atomic check on a guard variable (typically a single byte). On x86, this is a single `cmpb $0` + conditional jump — essentially free. The mutex is only used during the first call.

```asm
; Typical x86 codegen for Meyer's singleton:
  cmpb $0, guard_variable    ; atomic byte check
  jne  .already_initialized  ; fast path — just return
  ; slow path — call __cxa_guard_acquire, construct, __cxa_guard_release
.already_initialized:
  lea  rax, [static_instance]
  ret
```

<br>

#### Q2: "Should I return a pointer or a reference from `getInstance()`?"

Return a **reference**. A pointer implies the object might be null and invites callers to `delete` it:

```cpp
// BAD — pointer
static Singleton* getInstance();
auto* s = Singleton::getInstance();
delete s;  // compiles, disastrous

// GOOD — reference
static Singleton& getInstance();
auto& s = Singleton::getInstance();
// delete &s;  // possible but obviously wrong — code review will catch it
```

<br>

#### Q3: "Can I have a singleton template?"

Yes, using CRTP:

```cpp
template<typename Derived>
class Singleton {
public:
    static Derived& getInstance() {
        static Derived instance;
        return instance;
    }

    Singleton(const Singleton&) = delete;
    Singleton& operator=(const Singleton&) = delete;

protected:
    Singleton() = default;
    ~Singleton() = default;
};

class Logger : public Singleton<Logger> {
    friend class Singleton<Logger>;  // so base can call private constructor
    Logger() { std::cout << "Logger created\n"; }
public:
    void log(const std::string& msg) { std::cout << msg << "\n"; }
};

Logger::getInstance().log("hello");
```

<br>

#### Q4: "Is `std::call_once` faster or slower than Meyer's singleton?"

Roughly the same. Both use an atomic fast-path check. Meyer's uses the compiler's `__cxa_guard` machinery; `call_once` uses `pthread_once` (on POSIX) or similar. In practice, the difference is unmeasurable. Choose based on API needs, not performance.

<br>

#### Q5: "What about singletons in shared libraries (.so / .dll)?"

Each shared library gets its own copy of static variables. If your singleton is in a header-only library included by both the main executable and a `.so`, you may get **two instances**. Solutions:
- Put the singleton's `getInstance()` definition in a `.cpp` file compiled into exactly one library
- Use symbol visibility attributes to ensure one definition

<br><br>

---

## Reflection Questions

1. Why is Meyer's singleton preferred in modern C++?
2. What problem does double-checked locking try to solve? Why did it fail before C++11?
3. When would you choose `call_once` over Meyer's singleton?
4. How do you handle the dead reference problem?
5. Why are singletons considered an anti-pattern by some? How do you mitigate the drawbacks?

---

## Interview Questions

1. "Implement a thread-safe singleton in C++. Explain why it's thread-safe."
2. "What is the static initialization order fiasco? How does Meyer's singleton avoid it?"
3. "What is double-checked locking? Why was it broken before C++11?"
4. "How does C++11 guarantee thread-safe initialization of local statics?"
5. "What is the dead reference problem? How do you solve it?"
6. "When should you NOT use a singleton? What alternatives exist?"
7. "How would you make a singleton testable?"
8. "What happens if two shared libraries each instantiate the same singleton?"

---

**Next**: Day 16 — Factory & Abstract Factory →
