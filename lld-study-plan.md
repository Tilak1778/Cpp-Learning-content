# 2-Month LLD Study Plan for Systems C++ Engineer

> **Pace**: ~1.5-2 hrs/weekday, ~3-4 hrs/weekend days.
> Each topic gets breathing room for deeper understanding and revision.

---

## Month 1: Foundations, Patterns & Concurrency

---

### Week 1: Memory Ownership & RAII

| Day | Topic | Learn | Build |
|-----|-------|-------|-------|
| [**D1 Mon**](days/day01-move-semantics-rule-of-5.md) | Move semantics & Rule of 5 | Lvalue vs rvalue, `std::move`, copy/move constructors, when compiler generates what | Write a `String` class with proper copy/move semantics, verify with print statements in each special member |
| [**D2 Tue**](days/day02-unique-ptr.md) | `unique_ptr` deep dive | Ownership transfer, custom deleters, `make_unique`, array specialization | Implement `UniquePtr<T>` from scratch with custom deleter support |
| [**D3 Wed**](days/day03-shared-ptr.md) | `shared_ptr` deep dive | Control block layout, ref counting, weak references, aliasing constructor | Implement `SharedPtr<T>` + `WeakPtr<T>` with a shared control block |
| [**D4 Thu**](days/day04-review-atomic-refcount.md) | Review & refine | Re-read your D2-D3 code, add edge cases, test with sanitizers | Add thread-safe ref counting to your `SharedPtr` using `std::atomic` |
| [**D5 Fri**](days/day05-raii-scope-guard.md) | RAII pattern | Scope guards, file handle wrappers, lock guards | Build a `ScopeGuard` class and an RAII `FileHandle` wrapper |
| [**D6 Sat**](days/day06-cow-pimpl.md) | Copy-on-write & Pimpl | COW strings, Pimpl for ABI stability, compilation firewall | Implement a **COW string** and a **Pimpl-wrapped** `DatabaseClient` class |
| **D7 Sun** | **Weekly review** | Revisit all code, write short notes on each pattern's trade-offs | Refactor any messy code, ensure all builds with `-Wall -Wextra -fsanitize=address` |

**Key Deep-Dives:**
- Read libc++ source for `shared_ptr` control block
- Understand `std::allocator_traits` and how STL containers use allocators
- Draw memory layout diagrams for your smart pointers

---

### Week 2: Memory Allocators

| Day | Topic | Learn | Build |
|-----|-------|-------|-------|
| **D8 Mon** | How `new`/`delete` work | `operator new` vs `malloc`, alignment (`alignas`, `alignof`), placement new | Override global `operator new`/`delete` to track allocations, log sizes |
| **D9 Tue** | Arena (Bump) allocator | Linear allocation, bulk reset, no individual free, great for frame-based work | Implement an **Arena Allocator** — allocate from a contiguous buffer, `reset()` frees everything |
| **D10 Wed** | Pool allocator | Fixed-size blocks, free list (intrusive linked list inside free blocks) | Implement a **Fixed-Size Pool Allocator**, benchmark vs `new`/`delete` in a loop |
| **D11 Thu** | STL allocator interface | `std::allocator_traits`, making your allocator work with `std::vector` | Make your pool allocator compatible with STL containers |
| **D12 Fri** | Review & benchmark | Profile allocator performance, understand fragmentation trade-offs | Write a benchmark comparing arena vs pool vs default allocator for 1M allocations |
| **D13 Sat** | Object pool pattern | Reuse expensive objects (threads, connections, buffers), checkout/return semantics | Build a generic `ObjectPool<T>` with `acquire()` / `release()` |
| **D14 Sun** | **Weekly review** | Draw memory layout diagrams for each allocator | Write a 1-page summary: when to use which allocator and why |

**Key Deep-Dives:**
- Study how jemalloc/tcmalloc differ from glibc malloc
- Understand memory fragmentation (internal vs external)
- Profile with `valgrind --tool=massif` or AddressSanitizer

---

### Week 3: Core Design Patterns

| Day | Topic | Learn | Build |
|-----|-------|-------|-------|
| **D15 Mon** | Singleton | Meyer's singleton, `std::call_once`, why double-checked locking is broken pre-C++11 | Implement 3 singleton variants, test thread safety with 10 threads racing to `getInstance()` |
| **D16 Tue** | Factory & Abstract Factory | Decoupling creation from usage, registration-based factory | Build a **Shape Factory** — register creators by string name, create objects dynamically |
| **D17 Wed** | Builder pattern | Step-by-step construction, fluent API, immutable result objects | Build a **QueryBuilder** — `.select()`, `.from()`, `.where()`, `.build()` returns a query string |
| **D18 Thu** | Observer pattern | Subject/observer, weak references to prevent dangling, unsubscribe safety | Build an **EventEmitter** — `on(event, callback)`, `emit(event, data)`, `off(event, id)` |
| **D19 Fri** | Strategy & Command | Runtime-swappable algorithms, command encapsulation for undo/redo | Build a **text editor** with command pattern — `Type`, `Delete`, `undo()`, `redo()` |
| **D20 Sat** | State Machine | States, transitions, guards, entry/exit actions | Build a **TCP connection state machine** — CLOSED, LISTEN, SYN_SENT, ESTABLISHED, etc. |
| **D21 Sun** | **Weekly review** | Review all patterns, identify which ones you'd combine in a real system | **Mini-project**: A small plugin system using factory + observer + strategy together |

**Key Deep-Dives:**
- Compare runtime polymorphism (virtual) vs compile-time (CRTP/templates) for each pattern
- Study how Qt's signal/slot mechanism implements observer pattern
- Think about ownership: who owns the observers? the commands? the states?

---

### Week 4: Concurrency Fundamentals

| Day | Topic | Learn | Build |
|-----|-------|-------|-------|
| **D22 Mon** | Threads & Mutexes | `std::thread`, `std::mutex`, `lock_guard`, `unique_lock`, `scoped_lock` | Build a **thread-safe counter** and a **coarse-grained locked hash map** |
| **D23 Tue** | Condition variables | Spurious wakeups, predicate waits, producer-consumer signaling | Implement a **Bounded Blocking Queue** — blocks on full/empty |
| **D24 Wed** | Semaphore & Barrier | Counting semaphore, latch, barrier — all from primitives | Implement `Semaphore` and `Barrier` using `mutex` + `condition_variable` |
| **D25 Thu** | Reader-writer lock | Shared vs exclusive, starvation scenarios, read-preferring vs write-preferring | Implement `ReadWriteLock` from scratch, test with concurrent readers + writers |
| **D26 Fri** | Deadlocks & debugging | Lock ordering, lock hierarchy, detecting with TSan, `std::lock` | Write an intentional deadlock, detect with `-fsanitize=thread`, fix it |
| **D27 Sat** | Futures & Promises | `std::future`, `std::promise`, `std::packaged_task`, `std::async` | Build an async file processor — submit file paths, get future results |
| **D28 Sun** | **Weekly review** | Run all concurrency code under ThreadSanitizer | Write notes on every concurrency bug you hit this week and how you fixed it |

**Key Deep-Dives:**
- Understand the difference between `lock_guard`, `unique_lock`, and `scoped_lock`
- Study priority inversion and how real-time systems handle it
- Read about `std::jthread` and cooperative cancellation in C++20

---

## Month 2: Advanced Concurrency, System Designs, IPC & Mock Interviews

---

### Week 5: Advanced Concurrency

| Day | Topic | Learn | Build |
|-----|-------|-------|-------|
| **D29 Mon** | Thread pool design | Worker threads, task queue, graceful shutdown, `submit()` returning futures | Implement a **Thread Pool** with configurable thread count and `submit<T>(callable) → future<T>` |
| **D30 Tue** | Thread pool refinement | Priority tasks, thread naming for debugging, exception propagation | Add **priority queue** support and **exception forwarding** to your thread pool |
| **D31 Wed** | Atomics & memory ordering | `std::atomic`, `relaxed`, `acquire/release`, `seq_cst`, CAS loops | Implement a **lock-free stack** (Treiber stack) with `compare_exchange_weak` |
| **D32 Thu** | Lock-free queue | SPSC ring buffer, Michael-Scott queue concepts, ABA problem | Implement a **lock-free SPSC ring buffer** (single-producer, single-consumer) |
| **D33 Fri** | Actor model | Message-passing concurrency, one thread per actor, typed mailboxes | Implement an **Actor** class — own thread, message queue, `send(msg)`, `onReceive(handler)` |
| **D34 Sat** | Parallel algorithms | Map-reduce pattern, parallel `for_each`, work partitioning strategies | Build a **parallel word frequency counter** — partition file into chunks, map-reduce with your thread pool |
| **D35 Sun** | **Weekly review** | Draw happens-before diagrams for your lock-free structures | Stress-test everything with 16+ threads, fix any races found by TSan |

**Key Deep-Dives:**
- Study `compare_exchange_weak` vs `compare_exchange_strong`
- Understand the ABA problem and how hazard pointers / epoch-based reclamation solve it
- Read about work-stealing schedulers (e.g., Intel TBB)

---

### Week 6: System Design Problems (Part 1)

| Day | Topic | Learn | Build |
|-----|-------|-------|-------|
| **D36 Mon** | Design a Logger | Log levels, formatting, sinks (console, file), rotation | Build a **Logger** — log levels, timestamps, configurable sinks |
| **D37 Tue** | Logger — async & thread-safe | Background flush thread, batched writes, graceful shutdown on destruction | Add **async logging** — log calls push to queue, background thread writes to sink |
| **D38 Wed** | Design an LRU Cache | `unordered_map` + doubly-linked list, O(1) get/put, eviction | Implement **LRU Cache** with `get(key)`, `put(key, value)`, eviction callback |
| **D39 Thu** | LRU Cache — thread-safe | Fine-grained vs coarse locking, read-heavy optimization | Make your LRU cache **thread-safe**, benchmark under concurrent access |
| **D40 Fri** | Design a Rate Limiter | Token bucket algorithm, sliding window counter, sliding window log | Implement **Token Bucket** rate limiter — `allow()` returns true/false |
| **D41 Sat** | Design a Timer/Scheduler | Min-heap of deadlines, one-shot vs recurring, cancel support | Implement a **Timer** — `schedule(callback, delay)`, `scheduleRepeating(cb, interval)`, `cancel(id)` |
| **D42 Sun** | **Weekly review** | For each design: draw class diagram, list 3 trade-offs, explain thread safety | Practice explaining Logger and LRU Cache designs verbally (10 min each, as if in interview) |

**Key Deep-Dives:**
- Study how spdlog implements async logging
- Understand cache eviction policies beyond LRU: LFU, ARC, CLOCK
- Compare token bucket vs leaky bucket rate limiting

---

### Week 7: System Design Problems (Part 2) + IPC/Networking

| Day | Topic | Learn | Build |
|-----|-------|-------|-------|
| **D43 Mon** | Design a Connection Pool | Resource lifecycle, max connections, timeout, health checks | Implement a generic **ConnectionPool** — `acquire()` with timeout, `release()`, periodic health check |
| **D44 Tue** | Design a KV Store | In-memory hash map, write-ahead log, crash recovery | Implement a **persistent KV store** — `get`/`put`/`delete`, WAL, `load()` on startup |
| **D45 Wed** | Sockets & TCP basics | `socket`/`bind`/`listen`/`accept`/`connect`, blocking I/O | Build a **simple TCP echo server** — accept connections, echo back messages |
| **D46 Thu** | Non-blocking I/O & event loop | `epoll`/`kqueue`, reactor pattern, fd multiplexing | Convert your echo server to **non-blocking** with `epoll`/`kqueue`, handle multiple clients |
| **D47 Fri** | Serialization | Endianness, struct packing, length-prefixed framing, TLV encoding | Build a **binary message codec** — serialize/deserialize message structs over TCP |
| **D48 Sat** | Shared memory IPC | `mmap`, `shm_open`, inter-process communication, ring buffer in shared memory | Build a **shared memory ring buffer** between two processes (producer and consumer) |
| **D49 Sun** | **Weekly review** | Review all system designs end-to-end | **Mini-project**: Connect KV store + logger + connection pool into a **mock database service** |

**Key Deep-Dives:**
- Study how Redis implements its event loop
- Understand `SO_REUSEADDR`, `TCP_NODELAY`, and other socket options
- Compare `epoll` (Linux) vs `kqueue` (macOS/BSD) APIs

---

### Week 8: Advanced C++ Idioms + Mock Interviews

| Day | Topic | Learn | Build |
|-----|-------|-------|-------|
| **D50 Mon** | CRTP | Static polymorphism, mixin classes, avoiding vtable overhead | Implement a **CRTP-based logging mixin** and a **static interface** pattern |
| **D51 Tue** | Type erasure | How `std::function` works internally, small-buffer optimization (SBO) | Implement `Function<R(Args...)>` from scratch with SBO |
| **D52 Wed** | Policy-based design & Concepts | Compile-time strategy via templates, C++20 concepts for constraints | Build a **policy-based smart pointer** — configurable ownership policy, threading policy |
| **D53 Thu** | **Mock interview #1** | Timed: 15 min design on paper → 45 min implement | Pick 2: Thread Pool, LRU Cache, Logger — no references allowed |
| **D54 Fri** | **Mock interview #2** | Focus on articulating trade-offs out loud while coding | Pick 2: Rate Limiter, SharedPtr, Timer — explain decisions as you code |
| **D55 Sat** | **Mock interview #3** | Full simulation: interviewer asks follow-ups, you adapt the design | Pick 2: KV Store, Memory Allocator, Connection Pool — handle "what if" questions |
| **D56 Sun** | **Final review & cheat sheet** | Create a 3-4 page reference sheet covering every design | Rate yourself 1-5 on each topic, re-study anything below 4 |

**Key Deep-Dives:**
- Study how Boost.TypeErasure and `std::any` implement type erasure
- Understand when CRTP is better than virtual dispatch (and when it isn't)
- Review all your mock interview solutions — identify patterns in your mistakes

---

## Daily Routine

```
[20 min]  Theory  — Read about the topic (cppreference, book chapter, blog post)
[60 min]  Build   — Implement from scratch in C++17/20
[15 min]  Test    — Write a driver, run with sanitizers
[15 min]  Reflect — 3 bullet points: what clicked, what was hard, what trade-offs exist
```

## Weekly Habits

- **Every Sunday**: Review the week, refactor messy code, update your notes
- **Every build**: Compile with `-Wall -Wextra -Wpedantic -fsanitize=address,thread,undefined`
- **After each design problem**: Write a short doc — class diagram, API, trade-offs, complexity

---

## Recommended Resources

| Resource | When to Use |
|----------|-------------|
| *C++ Concurrency in Action* — Anthony Williams | Weeks 4-5 |
| *The Linux Programming Interface* — Michael Kerrisk | Week 7 (IPC/sockets) |
| cppreference.com | Daily reference |
| Compiler Explorer (godbolt.org) | Verify codegen for atomics/lock-free code |
| *Effective Modern C++* — Scott Meyers | Weeks 1, 8 (move semantics, templates) |
| *Design Patterns* — GoF (skim, don't memorize) | Week 3 pattern motivation |

---

## Self-Assessment Scale

Rate yourself after each week:

| Score | Meaning |
|-------|---------|
| 1 | Can't explain it |
| 2 | Understand the concept, can't code it |
| 3 | Can code it with references open |
| 4 | Can code from scratch and explain trade-offs |
| 5 | Can teach it and handle curveball follow-ups |

**Goal**: Everything at **4+** by Day 56.

---

## Progress Checklist

### Month 1
- [ ] Week 1: Memory Ownership & RAII
- [ ] Week 2: Memory Allocators
- [ ] Week 3: Core Design Patterns
- [ ] Week 4: Concurrency Fundamentals

### Month 2
- [ ] Week 5: Advanced Concurrency
- [ ] Week 6: System Design Problems (Part 1)
- [ ] Week 7: System Design Problems (Part 2) + IPC/Networking
- [ ] Week 8: Advanced C++ Idioms + Mock Interviews
