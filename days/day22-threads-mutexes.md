# Day 22: Threads & Mutexes

[← Back to Study Plan](../lld-study-plan.md) | [← Day 20](day20-state-machine.md)

> **Time**: ~1.5-2 hours
> **Goal**: Concurrency starts here. Learn how to spawn work with `std::thread`, the iron rule that every thread must be `join`ed or `detach`ed before it dies, what a **data race** actually is (and why it is undefined behavior, not "just a wrong answer"), and how mutual exclusion fixes it. Master the RAII lock wrappers — `std::lock_guard`, `std::unique_lock`, `std::scoped_lock` — and when each is the right tool. Build a thread-safe counter and a coarse-grained locked hash map, then prove correctness under contention with ThreadSanitizer.

---
---

# PART 1: THREADS — STARTING AND ENDING WORK

---
---

<br>

<h2 style="color: #2980B9;">📘 22.1 What a Thread Is</h2>

A **thread of execution** is an independent flow of control inside your process. All threads in a process share the same address space — the same heap, the same globals, the same code — but each gets its **own stack** and its own register set (including the instruction pointer).

```
Process
 ┌───────────────────────────────────────────────┐
 │  Shared:  code   globals   heap                │
 │                                                 │
 │  Thread 0        Thread 1        Thread 2       │
 │  ┌────────┐      ┌────────┐      ┌────────┐    │
 │  │ stack  │      │ stack  │      │ stack  │    │
 │  │ regs   │      │ regs   │      │ regs   │    │
 │  │ IP     │      │ IP     │      │ IP     │    │
 │  └────────┘      └────────┘      └────────┘    │
 └───────────────────────────────────────────────┘
```

That shared address space is the entire reason concurrency is hard: two threads can touch the *same* memory at the *same* time. The stack being private is why local variables are usually safe; the heap and globals being shared is why everything else needs care.

<br>

#### Why threads at all?

| Reason | Example |
|--------|---------|
| **Parallelism** (use more cores) | Sum a huge array in 8 chunks on 8 cores |
| **Latency hiding** (overlap I/O with compute) | One thread waits on the network while another renders |
| **Responsiveness** | UI thread stays free while a worker crunches |
| **Independent activities** | A server handles many clients concurrently |

Parallelism needs multiple cores to go faster; concurrency (interleaving) is a structuring tool that helps even on a single core.

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 22.2 std::thread — Spawning Work</h2>

`std::thread` is the C++ standard wrapper over an OS thread. You hand it a callable plus arguments; it starts running **immediately**.

```cpp
#include <thread>
#include <iostream>

void worker(int id, const std::string& tag) {
    std::cout << "worker " << id << " says " << tag << "\n";
}

int main() {
    std::thread t1(worker, 1, "hello");   // starts running now
    std::thread t2([]{ std::cout << "lambda thread\n"; });

    t1.join();   // wait for t1 to finish
    t2.join();   // wait for t2 to finish
    return 0;
}
```

<br>

#### Arguments are COPIED by default — a classic trap

`std::thread` copies (or moves) its arguments into internal storage *before* the new thread runs. That means a reference parameter does **not** bind to your caller's variable unless you opt in with `std::ref`:

```cpp
void increment(int& counter) { ++counter; }

int n = 0;
std::thread t(increment, n);          // COMPILE ERROR-ish intent: passes a COPY → caller's n unchanged
                                       // (actually fails to compile because increment wants int&)
std::thread t(increment, std::ref(n)); // correct: binds to caller's n
t.join();
```

The fix is `std::ref(n)` / `std::cref(n)`. But beware: if the new thread outlives `n`, you now have a dangling reference and undefined behavior. Lifetime is *your* problem.

<br>

#### Don't dangle the callable's captures either

```cpp
std::thread make_bad() {
    int local = 42;
    return std::thread([&]{ std::cout << local; });  // BUG: local dies when make_bad returns
}
```

Capturing by reference into a thread that escapes the current scope is a dangling-reference bug. Capture by value (`[=]` or `[local]`) when in doubt.

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 22.3 join, detach, and the Iron Rule</h2>

Every `std::thread` object is in one of two states: **joinable** (it represents a running or finished-but-not-reaped thread) or not. The **iron rule**:

> A joinable `std::thread` that is destroyed calls `std::terminate()` and kills your whole program.

So before a `std::thread` goes out of scope you **must** do exactly one of:

```cpp
t.join();    // block until the thread finishes, then mark non-joinable
// OR
t.detach();  // sever ownership — thread runs on its own, you can never join it again
```

```
                t.join()           t finishes, this thread continues
   joinable ───────────────► (caller blocks) ───────────────► not joinable

                t.detach()
   joinable ───────────────► not joinable (background thread runs free)

   destroyed while joinable ──► std::terminate()  ☠
```

<br>

#### Why does destroying a joinable thread terminate?

The standard committee chose `terminate()` deliberately: silently `join`ing in the destructor could deadlock or block at a surprising point; silently `detach`ing could leave a thread running on local variables that are about to be destroyed. Both "helpful" defaults are dangerous, so the language forces *you* to decide.

<br>

#### detach is almost always the wrong default

A detached thread is a thread you can no longer observe, wait for, or stop. If it touches anything that gets destroyed (locals, the object that spawned it, even `std::cout` during shutdown) you have a race against process teardown. Prefer `join`. Reach for `detach` only for truly fire-and-forget, self-contained background work — and even then a managed thread pool is usually better.

<br>

#### join is not exception-safe by itself

```cpp
void risky() {
    std::thread t(work);
    do_something_that_may_throw();   // if this throws...
    t.join();                        // ...this line is skipped → t destroyed joinable → terminate()
}
```

The robust answer is RAII: a small guard that joins in its destructor (often called a `jthread`-style wrapper). C++20 added exactly this as **`std::jthread`**, which joins automatically on destruction (and also carries a cooperative `stop_token`).

```cpp
#include <thread>          // C++20
std::jthread jt(work);     // joins automatically when jt goes out of scope — no terminate, no leak
```

If you are on C++17, write the guard yourself (you will, in today's bonus challenge).

<br><br>

---
---

# PART 2: DATA RACES — THE CORE PROBLEM

---
---

<br>

<h2 style="color: #2980B9;">📘 22.4 What a Data Race Actually Is</h2>

The precise definition matters because the consequences are severe. A **data race** occurs when:

1. Two or more threads access the **same** memory location,
2. **at least one** access is a **write**, and
3. the accesses are **not ordered** by any synchronization (no happens-before relationship between them).

If all three hold, the C++ standard says your program has **undefined behavior**. Not "you get a stale value." Not "you lose an update." *Undefined behavior* — the compiler is allowed to assume races never happen, so it may reorder, cache in registers, tear writes, or produce output that makes no sequential sense at all.

```
Thread A           Thread B
--------           --------
read  count        read  count        both read 41
add 1                                  A computes 42
                   add 1               B computes 42
write count=42                         A writes 42
                   write count=42      B writes 42
                                       → final value 42, not 43.  ONE UPDATE LOST.
```

That "lost update" picture is the *benign* mental model. The real hazard is worse: a non-atomic `long long` write on a 32-bit path can **tear** into two halves, producing a value neither thread ever wrote.

<br>

#### `count++` is three operations

```
count++   ≡   tmp = count;   (load)
              tmp = tmp + 1; (add)
              count = tmp;   (store)
```

Any thread can be preempted between those steps. The increment is **not atomic** unless you make it so (with a mutex or `std::atomic`).

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 22.5 Why "It Worked on My Machine" Is Meaningless</h2>

Races are **timing-dependent**. The interleaving that loses an update might happen 1 time in 10 million on your laptop and 1 time in 100 under production load on a 64-core box. Worse, because a race is UB, an optimizing compiler may *legally* transform racy code into something that breaks even single-threaded reasoning:

```cpp
bool done = false;          // not atomic
// Thread 1: while (!done) {}            // compiler may hoist `done` into a register → infinite loop
// Thread 2: done = true;               // never observed by Thread 1
```

The compiler sees no synchronization touching `done`, assumes single-threaded semantics, loads `done` once, and spins forever. The fix is to make `done` an `std::atomic<bool>` (or guard it with a mutex) so the access carries the required ordering.

**Rule of thumb:** shared mutable data needs *either* a mutex *or* atomics. There is no third "I'll just be careful" option.

<br><br>

---
---

# PART 3: MUTEXES & LOCK WRAPPERS

---
---

<br>

<h2 style="color: #2980B9;">📘 22.6 std::mutex — Mutual Exclusion</h2>

A **mutex** (mutual exclusion) is a lock that at most one thread can hold at a time. The region of code between acquiring and releasing the lock is the **critical section**, and the standard guarantees that operations inside one thread's critical section *happen-before* the next thread's — that is what kills the race.

```cpp
#include <mutex>

std::mutex m;
int count = 0;

void increment() {
    m.lock();        // blocks until we own the mutex
    ++count;         // critical section — exclusive access
    m.unlock();      // release
}
```

Raw `lock()` / `unlock()` is correct here but **fragile**: if `++count` threw, or you added an early `return`, you would leak the lock and deadlock the program forever. You should almost never call `lock()`/`unlock()` by hand.

<br>

#### Mutexes are not recursive by default

If a thread that already holds `std::mutex` tries to `lock()` it again, that is **undefined behavior** (typically a self-deadlock). If you genuinely need re-entrant locking (a locked function calling another locked function on the same object), use `std::recursive_mutex` — but treat that need as a design smell to revisit first.

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 22.7 lock_guard — The Default RAII Lock</h2>

`std::lock_guard` locks in its constructor and unlocks in its destructor. It is the simplest, cheapest, most common way to hold a mutex. It cannot be moved, copied, or unlocked early — and that rigidity is a feature.

```cpp
void increment() {
    std::lock_guard<std::mutex> lk(m);   // locks here
    ++count;
}                                         // unlocks here, even if ++count threw
```

Use `lock_guard` whenever you want to "hold this one mutex for this whole scope." That is 80% of cases.

<br>

> C++17 lets you write `std::lock_guard lk(m);` without the `<std::mutex>` thanks to class template argument deduction (CTAD).

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 22.8 unique_lock — The Flexible Lock</h2>

`std::unique_lock` does everything `lock_guard` does, plus:

- **Deferred locking** (`std::defer_lock`) — construct now, lock later.
- **Early unlock / relock** — `lk.unlock()` then `lk.lock()` again.
- **Movability** — you can return it from a function or store it.
- **Use with condition variables** — `cv.wait(lk, pred)` *requires* a `unique_lock` because it must unlock and relock internally (Day 23).
- **Try / timed locking** — `try_lock`, `try_lock_for`.

```cpp
std::unique_lock<std::mutex> lk(m, std::defer_lock);  // not locked yet
// ... do unlocked setup ...
lk.lock();            // lock now
shared_work();
lk.unlock();          // release early — don't hold the lock during the slow part below
slow_unshared_io();
```

The flexibility costs a little: `unique_lock` carries a bool tracking whether it currently owns the mutex, so it is marginally heavier than `lock_guard`. Use `lock_guard` by default; reach for `unique_lock` when you need one of its extra abilities (most importantly, condition variables).

<br>

#### lock_guard vs unique_lock

| | `lock_guard` | `unique_lock` |
|---|---|---|
| Locks on construction | Yes | Optional (`defer_lock`) |
| Unlock early / relock | No | Yes |
| Movable | No | Yes |
| Works with condition variables | No | Yes |
| Overhead | Minimal | Tracks ownership flag |
| Default choice? | Yes | Only when you need its features |

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 22.9 scoped_lock — Locking Multiple Mutexes Safely</h2>

C++17's `std::scoped_lock` is the modern way to hold **one or more** mutexes. With several mutexes it uses a deadlock-avoidance algorithm (the same one as `std::lock`) so the acquisition order can never cause a classic AB-BA deadlock.

```cpp
std::mutex m1, m2;

void transfer(/* ... */) {
    std::scoped_lock lk(m1, m2);   // locks BOTH, deadlock-free, in one statement
    // ... move money between two accounts ...
}                                   // unlocks both
```

If you lock two mutexes by hand in different orders on different threads, you get a deadlock (the whole story of Day 26). `scoped_lock` makes that mistake structurally impossible. For a single mutex it behaves just like `lock_guard`, so on C++17 many people use `scoped_lock` as their everyday default.

| Wrapper | Mutexes | Superpower |
|---------|---------|------------|
| `lock_guard` | exactly one | simplest, lightest |
| `scoped_lock` | one or many | deadlock-free multi-lock (C++17) |
| `unique_lock` | one | defer / unlock / move / condition variables |

<br><br>

---
---

# PART 4: DESIGNING THREAD-SAFE CLASSES

---
---

<br>

<h2 style="color: #2980B9;">📘 22.10 The Encapsulation Principle</h2>

The cleanest thread-safe design keeps the mutex and the data it protects **private and together**, and exposes only operations that lock internally. Callers never see the mutex.

```cpp
class Counter {
    mutable std::mutex m_;   // mutable so const methods can lock
    long long value_ = 0;
public:
    void increment()      { std::lock_guard lk(m_); ++value_; }
    long long get() const { std::lock_guard lk(m_); return value_; }
};
```

`mutable` is needed because `get()` is logically `const` (it doesn't change the observable value) but must still lock the mutex, which is a non-const operation.

<br>

#### The leak-the-reference trap

Never return a reference or pointer to the protected data — the caller could then touch it without the lock:

```cpp
// BAD: caller can read/modify value_ with no lock held
long long& get() { std::lock_guard lk(m_); return value_; }
```

Return by value, or pass a callback that runs under the lock. The mutex must guard *every* access, with no escape hatch.

<br>

#### Beware "thread-safe operations" that aren't a safe sequence

Even if every method locks, *combining* them is not atomic:

```cpp
if (!map.contains(k)) map.insert(k, v);   // RACE: another thread can insert k between the two calls
```

Each call locks individually, but the gap between them is unprotected. The fix is a single compound operation (`insert_if_absent`) that holds the lock across the whole check-then-act. This is why granularity matters — design operations at the level callers actually need to be atomic.

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 22.11 Coarse-Grained vs Fine-Grained Locking</h2>

| | Coarse-grained | Fine-grained |
|---|---|---|
| **Design** | One mutex guards the whole structure | Many mutexes guard small pieces (e.g., per-bucket) |
| **Correctness** | Easy to reason about | Easy to get wrong (ordering, races across pieces) |
| **Contention** | High — every operation serializes | Low — operations on different pieces run in parallel |
| **When** | Start here, default | Only when profiling shows the single lock is the bottleneck |

Today's hash map uses **coarse-grained** locking: one mutex for the whole map. It is simple and obviously correct. Sharded/striped locking (one lock per bucket group) is a Day-25/concurrent-structures topic — don't reach for it until a profiler tells you to.

```
Coarse:                          Fine (striped):
 ┌──────────────────────┐         ┌────┐┌────┐┌────┐┌────┐
 │  one mutex            │         │m0  ││m1  ││m2  ││m3  │
 │  ┌────┬────┬────┬───┐ │         ├────┤├────┤├────┤├────┤
 │  │ b0 │ b1 │ b2 │ b3│ │         │ b0 ││ b1 ││ b2 ││ b3 │
 │  └────┴────┴────┴───┘ │         └────┘└────┘└────┘└────┘
 └──────────────────────┘         ops on b0 and b2 run in parallel
 every op serializes
```

<br><br>

---
---

# PART 5: BUILD EXERCISE

---
---

<br>

<h2 style="color: #2980B9;">📘 22.12 Exercise: Thread-Safe Counter and Locked Hash Map</h2>

Build two thread-safe components, then hammer them from many threads and verify the results are exactly correct. The point is to *feel* that a mutex turns a racy program into a correct one, and to confirm it with ThreadSanitizer.

<br>

#### Skeleton (header + impl in one file)

```cpp
// threadsafe.h
#pragma once
#include <mutex>
#include <list>
#include <vector>
#include <utility>
#include <optional>
#include <functional>
#include <cstddef>

// ── 1. Thread-safe counter ──────────────────────────────
class Counter {
    mutable std::mutex m_;
    long long value_ = 0;
public:
    void increment() {
        std::lock_guard<std::mutex> lk(m_);
        ++value_;
    }
    void add(long long n) {
        std::lock_guard<std::mutex> lk(m_);
        value_ += n;
    }
    long long get() const {
        std::lock_guard<std::mutex> lk(m_);
        return value_;
    }
};

// ── 2. Coarse-grained locked hash map ───────────────────
//   One mutex guards the entire table. Separate chaining for collisions.
template <typename K, typename V>
class LockedHashMap {
    mutable std::mutex m_;
    std::vector<std::list<std::pair<K, V>>> buckets_;
    std::size_t size_ = 0;
    std::hash<K> hash_;

    std::list<std::pair<K, V>>& bucketFor(const K& k) {
        return buckets_[hash_(k) % buckets_.size()];
    }
    const std::list<std::pair<K, V>>& bucketFor(const K& k) const {
        return buckets_[hash_(k) % buckets_.size()];
    }

public:
    explicit LockedHashMap(std::size_t numBuckets = 64)
        : buckets_(numBuckets) {}

    // Insert or overwrite.
    void put(const K& key, const V& value) {
        std::lock_guard<std::mutex> lk(m_);
        auto& chain = bucketFor(key);
        for (auto& kv : chain) {
            if (kv.first == key) { kv.second = value; return; }
        }
        chain.emplace_back(key, value);
        ++size_;
    }

    // Returns a COPY of the value (never a reference into the table).
    std::optional<V> get(const K& key) const {
        std::lock_guard<std::mutex> lk(m_);
        const auto& chain = bucketFor(key);
        for (const auto& kv : chain) {
            if (kv.first == key) return kv.second;
        }
        return std::nullopt;
    }

    bool erase(const K& key) {
        std::lock_guard<std::mutex> lk(m_);
        auto& chain = bucketFor(key);
        for (auto it = chain.begin(); it != chain.end(); ++it) {
            if (it->first == key) { chain.erase(it); --size_; return true; }
        }
        return false;
    }

    // Compound atomic op: increment the value at key (creating it if absent).
    // This must hold the lock across check-then-act.
    void incrementValue(const K& key) {
        std::lock_guard<std::mutex> lk(m_);
        auto& chain = bucketFor(key);
        for (auto& kv : chain) {
            if (kv.first == key) { ++kv.second; return; }
        }
        chain.emplace_back(key, V{1});
        ++size_;
    }

    std::size_t size() const {
        std::lock_guard<std::mutex> lk(m_);
        return size_;
    }
};
```

<br>

#### Test driver

```cpp
// main.cpp
#include "threadsafe.h"
#include <thread>
#include <vector>
#include <iostream>
#include <cassert>

constexpr int kThreads = 8;
constexpr int kPerThread = 100000;

void testCounter() {
    std::cout << "=== Thread-safe counter ===\n";
    Counter c;
    std::vector<std::thread> ts;
    for (int i = 0; i < kThreads; ++i)
        ts.emplace_back([&]{
            for (int j = 0; j < kPerThread; ++j) c.increment();
        });
    for (auto& t : ts) t.join();

    long long expected = static_cast<long long>(kThreads) * kPerThread;
    std::cout << "  expected=" << expected << " got=" << c.get() << "\n";
    assert(c.get() == expected);   // would FAIL if increment() were unlocked
    std::cout << "  OK\n\n";
}

void testMapConcurrentWrites() {
    std::cout << "=== Locked hash map: concurrent incrementValue ===\n";
    LockedHashMap<int, long long> map;
    std::vector<std::thread> ts;
    // Every thread bumps the same 10 keys repeatedly.
    for (int i = 0; i < kThreads; ++i)
        ts.emplace_back([&]{
            for (int j = 0; j < kPerThread; ++j) map.incrementValue(j % 10);
        });
    for (auto& t : ts) t.join();

    long long expectedPerKey = static_cast<long long>(kThreads) * kPerThread / 10;
    for (int k = 0; k < 10; ++k) {
        auto v = map.get(k);
        assert(v.has_value());
        assert(*v == expectedPerKey);
    }
    std::cout << "  each of 10 keys == " << expectedPerKey << "  OK\n\n";
}

void testMapReadersAndWriters() {
    std::cout << "=== Locked hash map: mixed put/get/erase ===\n";
    LockedHashMap<int, int> map;
    std::vector<std::thread> ts;
    for (int i = 0; i < kThreads; ++i)
        ts.emplace_back([&, i]{
            for (int j = 0; j < 50000; ++j) {
                int key = (i * 50000 + j) % 1000;
                if (j % 3 == 0)      map.put(key, j);
                else if (j % 3 == 1) (void)map.get(key);
                else                 (void)map.erase(key);
            }
        });
    for (auto& t : ts) t.join();
    std::cout << "  no crash, no race, final size=" << map.size() << "  OK\n\n";
}

int main() {
    testCounter();
    testMapConcurrentWrites();
    testMapReadersAndWriters();
    std::cout << "All assertions passed.\n";
    return 0;
}
```

<br>

#### Compile and run (correctness)

```bash
g++ -std=c++17 -Wall -Wextra -Wpedantic -O2 -pthread \
    -o day22 main.cpp && ./day22
```

#### Compile and run under ThreadSanitizer (proves no data race)

```bash
g++ -std=c++17 -Wall -Wextra -Wpedantic -g -fsanitize=thread -pthread \
    -o day22_tsan main.cpp && ./day22_tsan
```

TSan should report **nothing**. To see it catch a real race, temporarily replace `Counter::increment()`'s body with a bare `++value_;` (no lock) and rerun — TSan will print a "data race" report pinpointing the two unsynchronized accesses.

<br>

#### Expected output

```
=== Thread-safe counter ===
  expected=800000 got=800000
  OK

=== Locked hash map: concurrent incrementValue ===
  each of 10 keys == 80000  OK

=== Locked hash map: mixed put/get/erase ===
  no crash, no race, final size=...
  OK

All assertions passed.
```

<br>

#### Bonus Challenges

1. **Prove the race exists.** Make a second counter class `RacyCounter` with an unlocked `++value_`. Run the same test; observe `got < expected` and a TSan report. Note how the shortfall grows with thread count.

2. **`std::atomic` counter.** Replace the mutex in `Counter` with `std::atomic<long long>` and `fetch_add`. Benchmark all three (locked, racy, atomic) for 8 threads × 1M increments. Which is fastest? Why is the atomic version both correct *and* faster than the mutex here?

3. **Write a `JoinGuard`.** On C++17, write an RAII wrapper that stores a `std::thread` and `join()`s it in its destructor, so an exception between spawn and join can't trigger `terminate()`. Compare with C++20 `std::jthread`.

4. **`std::ref` experiment.** Write a function `void bump(int& x)` and launch a thread `std::thread(bump, n)` *without* `std::ref`. Explain the compiler error. Then fix it and explain the lifetime hazard if the thread outlives `n`.

5. **Compound atomicity.** Add a `getOrDefault(key, def)` and a racy two-call version `contains() ? get() : def`. Show with a stress test that the two-call version can observe a torn intermediate state (a key present in `contains` but erased before `get`), while the single locked method cannot.

6. **Striped locking.** Upgrade `LockedHashMap` to hold an array of N mutexes, locking only `mutexes_[hash % N]` per operation. Benchmark against the coarse version under 8 threads on disjoint key ranges. Measure the throughput improvement — and find the one operation (`size()`) that becomes awkward.

<br><br>

---
---

# PART 6: DEEP DIVE — COMMON QUESTIONS

---
---

<br>

<h2 style="color: #2980B9;">📘 22.13 Q&A</h2>

<br>

#### Q1: "join vs detach — which do I use?"

Default to `join`. It is observable, it is safe, and it makes thread lifetimes explicit. Use `detach` only for genuinely fire-and-forget background work that owns all of its own state and never touches anything that might be destroyed. In modern code, prefer `std::jthread` (auto-joins) or a thread pool over manual `detach`.

<br>

#### Q2: "Why does destroying a joinable thread call terminate() instead of just joining?"

Because both "helpful" defaults are traps. Auto-`join` in the destructor could block indefinitely at an unexpected line (e.g., during stack unwinding from an exception), turning a logic bug into a hang. Auto-`detach` could leave a thread running on local variables that are about to vanish. The committee made the dangerous-to-ignore case loud (`terminate`) so you are forced to choose explicitly.

<br>

#### Q3: "lock_guard vs unique_lock vs scoped_lock — when each?"

- `lock_guard`: one mutex, whole scope, no frills. Your everyday default.
- `scoped_lock`: one *or more* mutexes; with several it is deadlock-free. Many use it as the C++17 default.
- `unique_lock`: when you need to unlock early, defer locking, move the lock, or use a condition variable (`cv.wait` requires it).

If you don't need a special power, use the simplest one.

<br>

#### Q4: "Is `std::atomic<int>` always better than a mutex for a counter?"

For a single counter that you only ever increment/read, yes — `fetch_add` is one lock-free instruction on most CPUs and beats a mutex. But atomics protect *one variable at a time*. The moment you need two pieces of state to change together atomically (e.g., update a value *and* a timestamp), a single atomic can't express that — you need a mutex (or a more complex lock-free design). Atomics aren't a free upgrade; they're a narrower tool.

<br>

#### Q5: "My class locks every method. Is it thread-safe?"

It is *internally* consistent, but composing two of its methods is not atomic. `if (m.contains(k)) m.get(k)` can race because another thread can erase `k` in the gap. True thread safety means designing operations at the granularity the caller needs to be atomic — sometimes a single compound method (`getOrInsert`, `incrementValue`) rather than two locked calls.

<br>

#### Q6: "Why `mutable std::mutex`?"

A logically-`const` method (a getter) still needs to lock, and locking mutates the mutex's internal state. Marking the mutex `mutable` lets `const` methods lock it without lying about their const-ness. This is the textbook legitimate use of `mutable`.

<br>

#### Q7: "Can I lock the same `std::mutex` twice in one thread?"

No — that is undefined behavior (usually self-deadlock). If a locked function legitimately needs to call another locked function on the same object, either refactor so the lock is taken once at the top (private unlocked helpers do the work), or use `std::recursive_mutex`. Prefer the refactor; recursive locking often hides a design issue.

<br>

#### Q8: "Why does an unlocked `bool done` flag spin forever even though another thread sets it true?"

Because the access is a data race (UB), the optimizer assumes single-threaded semantics, loads `done` into a register once, and never reloads it inside the loop. Nothing in the code tells it the value can change concurrently. Make `done` a `std::atomic<bool>` (or guard it with a mutex + condition variable) so the read carries the required memory ordering and is re-read each iteration.

<br><br>

---

## Reflection Questions

1. State the three conditions for a data race. Why is the consequence "undefined behavior" rather than merely "a wrong value"?
2. Why must every `std::thread` be joined or detached before destruction? What happens if you forget?
3. When would you choose `unique_lock` over `lock_guard`? Give a concrete scenario where `lock_guard` cannot work.
4. Why is "every method locks" insufficient for true thread safety? Give an example of a racy composition.
5. When is `std::atomic` the right tool and when do you still need a mutex?
6. Why does an optimizing compiler break an unsynchronized `while(!done){}` loop, and how do atomics fix it?

---

## Interview Questions

1. "What is a data race? How is it different from a race condition?"
2. "Implement a thread-safe counter. Now make it lock-free. Compare performance."
3. "Why does `std::thread`'s destructor call `terminate()` if the thread is joinable?"
4. "Explain the difference between `lock_guard`, `unique_lock`, and `scoped_lock`."
5. "Why does `std::thread` copy its arguments? How do you pass a reference, and what's the risk?"
6. "Design a thread-safe hash map. What does coarse-grained vs fine-grained locking mean, and when would you switch?"
7. "Walk through why `count++` is not atomic. Show the interleaving that loses an update."
8. "Why does `while(!done){}` with a plain `bool done` hang even after another thread sets `done = true`?"
9. "What is `std::jthread` and what two problems does it solve over `std::thread`?"
10. "When is `std::recursive_mutex` appropriate, and why is needing it often a design smell?"

---

**Next**: Day 23 — Condition Variables →
