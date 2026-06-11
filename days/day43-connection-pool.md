# Day 43: Design a Connection Pool

[← Back to Study Plan](../lld-study-plan.md) | [← Day 41](day41-timer-scheduler.md)

> **Time**: ~1.5-2 hours
> **Goal**: A connection pool is the canonical resource-management problem in backend systems: connections (to a DB, a cache, a remote service) are **expensive to create** and **scarce**, so we keep a bounded set of them alive and hand them out on demand. Learn resource lifecycle (create → idle → in-use → idle → evict), `max` connection caps, **acquire with timeout** (block when exhausted, fail gracefully), **idle eviction** (return memory/file-descriptors when load drops), **health checks** (never hand out a dead connection), and an **RAII handle** that returns the connection to the pool on destruction. Build a generic, thread-safe `ConnectionPool` with `acquire(timeout)`, `release()`, and a background reaper.

---
---

# PART 1: WHY A CONNECTION POOL EXISTS

---
---

<br>

<h2 style="color: #2980B9;">📘 43.1 The Core Problem</h2>

Creating a connection to a remote system is one of the most expensive things a server does on the hot path:

| Step | Typical cost |
|------|--------------|
| TCP 3-way handshake (1 RTT) | 0.5–50 ms depending on distance |
| TLS handshake (1–2 RTT + asymmetric crypto) | 5–100 ms |
| Auth / login (DB password check, often bcrypt) | 10–250 ms |
| Session setup (set search_path, timezone, prepared statements) | 1–10 ms |

A single web request that opens a fresh DB connection, runs one 2 ms query, and closes it can spend **200 ms** on connection setup and **2 ms** on actual work. Under load this collapses: each open connection consumes a file descriptor, a kernel socket buffer, and (on the DB side) a backend process or thread. Postgres defaults to `max_connections = 100`; open 101 sockets and the 101st request gets refused.

The connection pool solves both problems at once:

> "Keep a small bounded set of already-established connections alive. Lend one to a caller, take it back when they're done, and never exceed the cap."

```
Without pool:                        With pool (max=4):
  req1: connect(200ms) query close     startup: open 4 connections once
  req2: connect(200ms) query close     req1: acquire(0.01ms) query release(0.01ms)
  req3: connect(200ms) query close     req2: acquire(0.01ms) query release(0.01ms)
  ...                                   req5: acquire BLOCKS until one frees up
  → 200ms tax per request              → ~0 tax per request, hard cap on resources
```

<br>

This is the **Object Pool** pattern (Day 13) specialized for connections, with three extra concerns that don't show up for plain objects: connections can **die** silently (server restart, idle timeout, network blip), they're **scarce** (so we must block/timeout rather than grow infinitely), and they should be **evicted when idle** to release resources the server doesn't currently need.

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 43.2 The Connection Lifecycle</h2>

Every pooled connection moves through a small state machine (this is a real FSM — see Day 20):

```
         create()                acquire()
   ┌────────────┐  success   ┌──────────┐ check out  ┌──────────┐
   │  CREATING  │ ─────────► │   IDLE   │ ──────────►│ IN_USE   │
   └────────────┘            │ (in pool)│            │(borrowed)│
         │ fail              └──────────┘ ◄──────────└──────────┘
         ▼                        │  ▲     release()
   ┌──────────┐  idle too long    │  │ health-check OK
   │  FAILED  │ ◄─────────────────┘  │
   └──────────┘  health-check FAIL   │
         │                           │
         └──── destroy() ────────────┘ (closed, fd released)
```

- **IDLE** — alive, sitting in the pool's free list, available for `acquire()`.
- **IN_USE** — checked out by exactly one caller. Must not be touched by anyone else.
- **destroyed** — closed and removed, either because it failed a health check or was evicted for being idle too long.

The pool's invariant: `created == idle + in_use` at all times, and `created <= maxSize`.

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 43.3 The Four Hard Decisions</h2>

| Decision | Options | Recommended default |
|----------|---------|---------------------|
| **What happens when exhausted?** | grow unbounded / fail immediately / **block with timeout** | block with timeout — fail-fast but tolerate spikes |
| **When to create connections?** | lazily on first `acquire` / **pre-warm `minSize` at startup** | pre-warm `minSize`, grow on demand to `maxSize` |
| **How to detect dead connections?** | trust them / **validate on borrow** / validate on return / background ping | validate on borrow (cheap) + background reaper |
| **When to close idle connections?** | never / **after idleTimeout, down to `minSize`** | reaper closes idle conns older than `idleTimeout`, keeps `minSize` warm |

The combination "pre-warm `minSize`, grow to `maxSize`, block-with-timeout on exhaustion, validate-on-borrow, reap idle beyond `minSize`" is essentially what HikariCP, pgbouncer, and the Go `database/sql` pool all do.

<br><br>

---
---

# PART 2: ACQUIRE, RELEASE, AND TIMEOUTS

---
---

<br>

<h2 style="color: #2980B9;">📘 43.4 Acquire With Timeout — the Condition Variable Pattern</h2>

When all connections are checked out and we're at `maxSize`, a new `acquire()` must wait. The wrong way is a spin loop. The right way is a `condition_variable`:

```cpp
PooledHandle acquire(std::chrono::milliseconds timeout) {
    std::unique_lock<std::mutex> lock(mtx_);

    const auto deadline = std::chrono::steady_clock::now() + timeout;

    while (true) {
        // 1. Is there an idle connection ready to hand out?
        if (!idle_.empty()) {
            Conn* c = idle_.back();
            idle_.pop_back();
            // ... (validate before returning — see §43.6)
            return PooledHandle(this, c);
        }
        // 2. Room to create a brand-new one?
        if (created_ < maxSize_) {
            ++created_;
            lock.unlock();              // creation is slow — do NOT hold the lock
            Conn* c = factory_();       // may throw / block on TCP+TLS
            lock.lock();
            return PooledHandle(this, c);
        }
        // 3. Pool is full and all in use — wait for a release() or timeout.
        if (cv_.wait_until(lock, deadline) == std::cv_status::timeout) {
            throw PoolTimeout("acquire timed out");
        }
        // woke up: loop and re-check (spurious wakeups + races handled by while)
    }
}
```

Three subtle but critical points:

1. **Never call the factory while holding the lock.** Creating a connection can take 200 ms; holding the mutex that long serializes the entire pool. Bump `created_` *before* unlocking (so two threads don't both decide there's room), then create outside the lock.
2. **Use `wait_until`, not `wait_for`, inside a loop.** `wait_for(remaining)` recomputed each iteration drifts on every spurious wakeup; a fixed `deadline` does not. The `while(true)` re-checks the predicate after every wake — this is mandatory because `wait` can return spuriously.
3. **`release()` must `notify_one()`** so a waiter wakes up. If you have many waiters and return many connections, `notify_all()` avoids lost-wakeup edge cases at the cost of a thundering herd.

<br>

```
Timeline of an exhausted pool (max=2):

  T1: acquire() ─► gets conn A         [idle: -, in_use: A]
  T2: acquire() ─► gets conn B         [idle: -, in_use: A,B]
  T3: acquire() ─► cv_.wait_until(...)  ── BLOCKED ──┐
  T1: release(A) ─► idle.push(A); notify_one() ──────┘ wakes T3
  T3: re-checks predicate ─► idle not empty ─► gets A
```

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 43.5 Release and the RAII Handle</h2>

Manual `release()` is a leak waiting to happen — an early `return`, a thrown exception, or a forgotten code path strands a connection in `IN_USE` forever (a "connection leak", the most common production pool bug). The fix is the same RAII handle pattern from Day 13: a move-only wrapper whose destructor returns the connection.

```cpp
class PooledHandle {
    ConnectionPool* pool_ = nullptr;
    Conn*           conn_ = nullptr;
public:
    PooledHandle() = default;
    PooledHandle(ConnectionPool* p, Conn* c) : pool_(p), conn_(c) {}

    ~PooledHandle() { if (conn_) pool_->release(conn_); }

    PooledHandle(const PooledHandle&)            = delete;
    PooledHandle& operator=(const PooledHandle&) = delete;

    PooledHandle(PooledHandle&& o) noexcept
        : pool_(o.pool_), conn_(o.conn_) { o.conn_ = nullptr; }

    PooledHandle& operator=(PooledHandle&& o) noexcept {
        if (this != &o) {
            if (conn_) pool_->release(conn_);
            pool_ = o.pool_; conn_ = o.conn_; o.conn_ = nullptr;
        }
        return *this;
    }

    Conn* operator->() const { return conn_; }
    Conn& operator*()  const { return *conn_; }
    explicit operator bool() const { return conn_ != nullptr; }
};
```

The destructor runs no matter how the scope exits:

```cpp
void handleRequest(ConnectionPool& pool) {
    auto h = pool.acquire(50ms);
    h->execute("SELECT 1");
    if (somethingWrong) throw std::runtime_error("oops"); // h still returned!
    // ... normal path ...
}   // h destructs here → release() → connection back in pool
```

<br>

A common variant returns a `std::shared_ptr<Conn>` with a custom deleter instead of a bespoke handle — useful when the surrounding API already speaks `shared_ptr`:

```cpp
std::shared_ptr<Conn> acquireShared(std::chrono::milliseconds t) {
    PooledHandle h = acquire(t);
    Conn* raw = h.release();   // detach from handle
    return std::shared_ptr<Conn>(raw, [this](Conn* c){ this->release(c); });
}
```

The trade-off is the control-block allocation and refcount overhead vs the zero-overhead move-only handle.

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 43.6 Health Checks — Never Hand Out a Corpse</h2>

Connections die silently. The DB restarts, an idle-timeout on the server side closes the socket, a load balancer drops a long-lived flow, a NAT table forgets the mapping. The socket on our side still *looks* open — `send()` may even succeed into the kernel buffer — but the next round-trip will fail. If we hand such a connection to a caller, their query fails with a confusing error that has nothing to do with their code.

Three places to validate:

| Strategy | When | Cost | Notes |
|----------|------|------|-------|
| **Validate on borrow** | inside `acquire()`, before returning | one cheap round-trip (or a fast local check) | Most common. Adds latency to every acquire. |
| **Validate on return** | inside `release()` | one round-trip per release | Keeps the pool "clean" but taxes the return path. |
| **Background ping** | reaper thread, periodically | amortized, off the hot path | Best for latency, but a conn can still die between pings. |

The validation itself should be the cheapest possible signal:

```cpp
bool isAlive(Conn* c) {
    auto now = std::chrono::steady_clock::now();
    // 1. Cheap heuristic: was it used very recently? Trust it.
    if (now - c->lastUsed < std::chrono::milliseconds(500)) return true;
    // 2. Otherwise do a real liveness probe (DB: "SELECT 1"; socket: zero-byte
    //    non-blocking recv to detect EOF/RST). Must time out fast.
    return c->ping();
}
```

If `isAlive` fails inside `acquire()`, **destroy the corpse and try again** (don't return it):

```cpp
if (!isAlive(c)) {
    destroy(c);          // close fd, --created_
    --created_;
    continue;            // loop back: maybe create a fresh one or grab another idle
}
```

This is why §43.4's body is a `while(true)` loop — a failed health check just continues the loop.

<br><br>

---
---

# PART 3: IDLE EVICTION & THE REAPER

---
---

<br>

<h2 style="color: #2980B9;">📘 43.7 Why Evict Idle Connections</h2>

A pool that grew to `maxSize` during a traffic spike will sit on `maxSize` connections forever unless something closes them. Each idle connection:

- Holds a file descriptor on both ends (fds are a finite per-process resource).
- Consumes a backend process/thread on the DB server (Postgres) — wasted capacity other clients can't use.
- May be silently killed by the server's own idle-timeout, becoming a landmine for the next borrower.

So we run a **reaper**: a background thread that periodically scans the idle list and closes connections that have been idle longer than `idleTimeout`, **but never below `minSize`** (keep a warm baseline so the next request doesn't pay creation cost).

```
idle list, idleTimeout = 30s, minSize = 2, now = 12:00:30

  conn   idleSince   age    action
  ────   ─────────   ───    ──────────────────────
  A      11:59:55    35s    KEEP (would drop below minSize... check count)
  B      12:00:10    20s    keep (age < idleTimeout)
  C      12:00:00    30s    EVICT (age >= idleTimeout, count above minSize)
  D      11:59:50    40s    EVICT
  → after reap: idle = {A, B}, created drops by 2
```

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 43.8 Reaper Implementation</h2>

The reaper is a thread that sleeps on a `condition_variable` with a timeout (so it can be woken to shut down promptly) and reaps on each tick:

```cpp
void reaperLoop() {
    std::unique_lock<std::mutex> lock(mtx_);
    while (!stopping_) {
        // Sleep until next sweep OR until shutdown signal.
        shutdownCv_.wait_for(lock, sweepInterval_, [this]{ return stopping_; });
        if (stopping_) break;

        const auto now = std::chrono::steady_clock::now();
        // Walk idle list; evict stale ones but keep at least minSize alive.
        auto it = idle_.begin();
        while (it != idle_.end() && created_ > minSize_) {
            Conn* c = *it;
            if (now - c->lastUsed >= idleTimeout_) {
                it = idle_.erase(it);
                destroy(c);          // close socket, free Conn
                --created_;
            } else {
                ++it;
            }
        }
        // (optional) top up back to minSize if we fell below due to deaths
    }
}
```

Note the reaper holds `mtx_` while scanning but `destroy()` here is just `close(fd)` + `delete` — fast and local, so holding the lock is fine. If `destroy()` could block (e.g., a graceful protocol-level shutdown handshake), collect the victims under the lock, release it, then close them outside.

<br>

#### Lifetime and shutdown

The reaper must be joined cleanly in the pool's destructor:

```cpp
~ConnectionPool() {
    {
        std::lock_guard<std::mutex> lock(mtx_);
        stopping_ = true;
    }
    shutdownCv_.notify_all();
    if (reaper_.joinable()) reaper_.join();   // join BEFORE destroying members it touches
    for (Conn* c : idle_) destroy(c);          // close everything still pooled
}
```

A classic bug: forgetting to join the reaper, so it touches `idle_`/`mtx_` after they've been destroyed → use-after-free. Always join background threads before the fields they reference are torn down.

<br><br>

---
---

# PART 4: THREAD SAFETY & FAIRNESS

---
---

<br>

<h2 style="color: #2980B9;">📘 43.9 Concurrency Model</h2>

A connection pool is shared by every request thread, so it is inherently concurrent. The minimal correct model:

- **One `std::mutex`** protects `idle_`, `created_`, and `stopping_`.
- **One `condition_variable`** for waiters in `acquire()`, notified by `release()`.
- **Connection creation runs outside the lock** (it's the slow part).
- **The borrowed connection itself is NOT pool-synchronized** — exactly one caller holds it, so they use it single-threaded. The pool only synchronizes the *transfer* of ownership.

```
        ┌─────────────────── ConnectionPool ───────────────────┐
        │  mutex ── protects ── idle_, created_, stopping_      │
        │  cv    ── acquire() waiters wait; release() notifies  │
        └───────────────────────────────────────────────────────┘
              ▲ acquire/release (locked)        ▲ reaper tick (locked)
   ┌──────────┴────────┐ ┌────────┴──────────┐ ┌┴───────────┐
   │ request thread 1  │ │ request thread 2  │ │ reaper     │
   │ uses Conn A alone │ │ uses Conn B alone │ │ thread     │
   └───────────────────┘ └───────────────────┘ └────────────┘
```

<br>

#### Fairness

Plain `notify_one()` gives no FIFO guarantee — the OS picks which waiter wakes, so a thread can starve under sustained load. If fairness matters, give each waiter a ticket and a per-waiter `condition_variable` (or a `std::queue` of `std::condition_variable*`), and on `release()` wake the **oldest** waiter. Most pools accept the unfairness because under healthy sizing nobody waits long.

<br><br>

---
---

# PART 5: BUILD EXERCISE

---
---

<br>

<h2 style="color: #2980B9;">📘 43.10 Exercise: A Generic Thread-Safe Connection Pool</h2>

Build a `ConnectionPool` parameterized over a connection type, with: pre-warmed `minSize`, on-demand growth to `maxSize`, `acquire(timeout)` that blocks on exhaustion, an RAII handle, validate-on-borrow health checks, and a background reaper that evicts idle connections beyond `minSize`. We simulate a "connection" rather than open real sockets so it builds anywhere (real socket creation comes Day 45).

> **Portability note**: This file uses only `<thread>`, `<mutex>`, `<condition_variable>`, and `<chrono>` — fully portable to Linux and macOS. The factory is where you'd plug in real `connect()`/TLS code later.

<br>

#### Skeleton

```cpp
// connection_pool.h
#pragma once
#include <atomic>
#include <chrono>
#include <condition_variable>
#include <functional>
#include <iostream>
#include <memory>
#include <mutex>
#include <stdexcept>
#include <thread>
#include <vector>

using Clock    = std::chrono::steady_clock;
using Millis   = std::chrono::milliseconds;

struct PoolTimeout : std::runtime_error {
    explicit PoolTimeout(const char* m) : std::runtime_error(m) {}
};

// ── A fake "connection" so the demo builds with no real network. ──
struct FakeConn {
    int        id;
    bool       healthy = true;          // flip to simulate a dead connection
    Clock::time_point lastUsed = Clock::now();

    explicit FakeConn(int i) : id(i) {
        std::cout << "  [CREATE] conn " << id << " (expensive: handshake+auth)\n";
    }
    ~FakeConn() { std::cout << "  [CLOSE ] conn " << id << "\n"; }

    bool ping() const { return healthy; }                 // real impl: SELECT 1
    void exec(const std::string& q) {
        std::cout << "  conn " << id << " exec: " << q << "\n";
    }
};

template <typename Conn>
class ConnectionPool;

// ── RAII handle: returns the connection on destruction. ──
template <typename Conn>
class PooledHandle {
    ConnectionPool<Conn>* pool_ = nullptr;
    Conn*                 conn_ = nullptr;
public:
    PooledHandle() = default;
    PooledHandle(ConnectionPool<Conn>* p, Conn* c) : pool_(p), conn_(c) {}
    ~PooledHandle() { if (conn_) pool_->release(conn_); }

    PooledHandle(const PooledHandle&)            = delete;
    PooledHandle& operator=(const PooledHandle&) = delete;
    PooledHandle(PooledHandle&& o) noexcept : pool_(o.pool_), conn_(o.conn_) { o.conn_ = nullptr; }
    PooledHandle& operator=(PooledHandle&& o) noexcept {
        if (this != &o) { if (conn_) pool_->release(conn_);
            pool_ = o.pool_; conn_ = o.conn_; o.conn_ = nullptr; }
        return *this;
    }
    Conn* operator->() const { return conn_; }
    Conn& operator*()  const { return *conn_; }
    Conn* release()          { Conn* c = conn_; conn_ = nullptr; return c; }
    explicit operator bool() const { return conn_ != nullptr; }
};

template <typename Conn>
class ConnectionPool {
public:
    struct Config {
        std::size_t minSize      = 2;
        std::size_t maxSize      = 8;
        Millis      idleTimeout  = Millis(30000);
        Millis      sweep        = Millis(5000);
    };

    ConnectionPool(std::function<Conn*()> factory, Config cfg)
        : factory_(std::move(factory)), cfg_(cfg)
    {
        std::lock_guard<std::mutex> lk(mtx_);
        for (std::size_t i = 0; i < cfg_.minSize; ++i) {   // pre-warm
            idle_.push_back(factory_());
            ++created_;
        }
        reaper_ = std::thread([this]{ reaperLoop(); });
    }

    ~ConnectionPool() {
        { std::lock_guard<std::mutex> lk(mtx_); stopping_ = true; }
        shutdownCv_.notify_all();
        if (reaper_.joinable()) reaper_.join();
        for (Conn* c : idle_) delete c;                    // dtor prints [CLOSE]
        // (in-use conns are caller-owned via handles; assume returned by now)
    }

    PooledHandle<Conn> acquire(Millis timeout) {
        std::unique_lock<std::mutex> lk(mtx_);
        const auto deadline = Clock::now() + timeout;
        while (true) {
            if (!idle_.empty()) {
                Conn* c = idle_.back();
                idle_.pop_back();
                if (!validate(c)) { delete c; --created_; continue; }
                return PooledHandle<Conn>(this, c);
            }
            if (created_ < cfg_.maxSize) {
                ++created_;
                lk.unlock();
                Conn* c = nullptr;
                try { c = factory_(); }
                catch (...) { lk.lock(); --created_; cv_.notify_one(); throw; }
                lk.lock();
                return PooledHandle<Conn>(this, c);
            }
            if (cv_.wait_until(lk, deadline) == std::cv_status::timeout)
                throw PoolTimeout("acquire timed out");
        }
    }

    void release(Conn* c) {
        {
            std::lock_guard<std::mutex> lk(mtx_);
            c->lastUsed = Clock::now();
            idle_.push_back(c);
        }
        cv_.notify_one();
    }

    std::size_t created()   { std::lock_guard<std::mutex> lk(mtx_); return created_; }
    std::size_t available() { std::lock_guard<std::mutex> lk(mtx_); return idle_.size(); }

private:
    bool validate(Conn* c) {                               // health check on borrow
        if (Clock::now() - c->lastUsed < Millis(500)) return true;
        return c->ping();
    }

    void reaperLoop() {
        std::unique_lock<std::mutex> lk(mtx_);
        while (!stopping_) {
            shutdownCv_.wait_for(lk, cfg_.sweep, [this]{ return stopping_; });
            if (stopping_) break;
            const auto now = Clock::now();
            for (auto it = idle_.begin(); it != idle_.end() && created_ > cfg_.minSize; ) {
                if (now - (*it)->lastUsed >= cfg_.idleTimeout) {
                    Conn* c = *it;
                    it = idle_.erase(it);
                    delete c;                              // dtor prints [CLOSE]
                    --created_;
                } else ++it;
            }
        }
    }

    std::function<Conn*()> factory_;
    Config                 cfg_;
    std::vector<Conn*>     idle_;
    std::size_t            created_  = 0;
    bool                   stopping_ = false;
    std::mutex             mtx_;
    std::condition_variable cv_;          // acquire() waiters
    std::condition_variable shutdownCv_;  // reaper sleep / shutdown
    std::thread            reaper_;
};
```

<br>

#### Test driver

```cpp
// main.cpp
#include "connection_pool.h"
#include <cassert>

int main() {
    int nextId = 1;
    ConnectionPool<FakeConn>::Config cfg;
    cfg.minSize = 2; cfg.maxSize = 3;
    cfg.idleTimeout = Millis(300); cfg.sweep = Millis(100);

    std::cout << "=== 1. Pre-warm minSize ===\n";
    ConnectionPool<FakeConn> pool([&]{ return new FakeConn(nextId++); }, cfg);
    assert(pool.created() == 2);
    assert(pool.available() == 2);

    std::cout << "\n=== 2. Acquire/use/auto-release (RAII) ===\n";
    {
        auto h = pool.acquire(Millis(100));
        h->exec("SELECT 1");
        assert(pool.available() == 1);
    }   // h destructs -> released
    assert(pool.available() == 2);

    std::cout << "\n=== 3. Grow on demand to maxSize, then time out ===\n";
    {
        auto a = pool.acquire(Millis(100));
        auto b = pool.acquire(Millis(100));
        auto c = pool.acquire(Millis(100));   // grows to 3 (maxSize)
        assert(pool.created() == 3);
        try {
            auto d = pool.acquire(Millis(50)); // all in use -> timeout
            (void)d; assert(false && "should have thrown");
        } catch (const PoolTimeout& e) {
            std::cout << "  caught expected: " << e.what() << "\n";
        }
    }   // a,b,c released here

    std::cout << "\n=== 4. Dead connection is discarded on borrow ===\n";
    {
        auto h = pool.acquire(Millis(100));
        h->healthy = false;                    // simulate it dying while in use
        h->lastUsed = Clock::now() - Millis(1000); // force a real ping on next borrow
    }   // released back as 'dead'
    {
        auto h = pool.acquire(Millis(100));    // validate() pings -> false -> discard+recreate
        h->exec("after recovery");
    }

    std::cout << "\n=== 5. Reaper evicts idle beyond minSize ===\n";
    {   // push created up to 3, release all, let reaper run
        auto a = pool.acquire(Millis(100));
        auto b = pool.acquire(Millis(100));
        auto c = pool.acquire(Millis(100));
        (void)a;(void)b;(void)c;
    }
    std::this_thread::sleep_for(Millis(600));   // > idleTimeout + a sweep
    std::cout << "  created after reap (expect minSize=2): " << pool.created() << "\n";
    assert(pool.created() == cfg.minSize);

    std::cout << "\n=== Done (pool destructs, reaper joins, conns close) ===\n";
    return 0;
}
```

<br>

#### Compile and run

```bash
g++ -std=c++17 -Wall -Wextra -Wpedantic -pthread -fsanitize=address,undefined \
    -o day43 main.cpp && ./day43
```

<br>

#### Expected output pattern

```
=== 1. Pre-warm minSize ===
  [CREATE] conn 1 (expensive: handshake+auth)
  [CREATE] conn 2 (expensive: handshake+auth)

=== 2. Acquire/use/auto-release (RAII) ===
  conn 2 exec: SELECT 1

=== 3. Grow on demand to maxSize, then time out ===
  [CREATE] conn 3 (expensive: handshake+auth)
  caught expected: acquire timed out

=== 4. Dead connection is discarded on borrow ===
  [CLOSE ] conn ...
  [CREATE] conn ...
  conn ... exec: after recovery

=== 5. Reaper evicts idle beyond minSize ===
  [CLOSE ] conn ...
  created after reap (expect minSize=2): 2

=== Done (pool destructs, reaper joins, conns close) ===
  [CLOSE ] conn ...
  [CLOSE ] conn ...
```

(Exact ids vary because `idle_` is a stack — the most-recently-released conn is reused first.)

<br>

#### Bonus Challenges

1. **Fair acquire**: replace `notify_one()` with a FIFO ticket queue so the longest-waiting thread is served first. Verify with a stress test that no thread starves.
2. **maxLifetime**: in addition to idle eviction, close any connection older than `maxLifetime` *even if busy* (after it's returned) — protects against server-side max-connection-age limits. HikariCP defaults this to 30 min.
3. **Validate-on-return**: move the health check into `release()` and compare the latency profile vs validate-on-borrow under a benchmark.
4. **Real sockets**: replace `FakeConn` with a TCP connection to `127.0.0.1` (use the Day 45 socket code). Implement `ping()` as a non-blocking zero-byte `recv()` that detects EOF/RST.
5. **Metrics**: expose `waitCount`, `waitTimeP99`, `borrowCount`, `timeoutCount`. Print them on shutdown. These are exactly the gauges HikariCP/pgbouncer surface.
6. **Graceful drain**: add `drain()` that stops handing out new connections, waits for all in-use to return (with a timeout), then closes everything — for clean rolling deploys.

<br><br>

---
---

# PART 6: DEEP DIVE — COMMON QUESTIONS

---
---

<br>

<h2 style="color: #2980B9;">📘 43.11 Q&A</h2>

<br>

#### Q1: "How is a connection pool different from a plain object pool (Day 13)?"

The pattern is identical; three concerns differ. (1) Connections **die silently**, so you need health checks — an object never "rots". (2) Connections are **scarce and capped**, so exhaustion must *block with a timeout* rather than grow unbounded. (3) Idle connections **cost real resources** (fds, server backends) and get killed by the server, so you **evict** them with a reaper. A plain object pool typically does none of these.

<br>

#### Q2: "Why block on exhaustion instead of just creating more connections?"

Because connections are backed by a hard external limit (e.g., Postgres `max_connections`). If every app instance grows its pool unboundedly during a spike, you exhaust the DB's connection slots and *every* client — including healthy ones — starts getting refused. Blocking with a timeout applies back-pressure: requests queue briefly, and if the spike is real, they fail fast with a clear "pool exhausted" error rather than overwhelming the database.

<br>

#### Q3: "Should I health-check on borrow, on return, or in the background?"

Validate-on-borrow is the safe default: it guarantees the caller never gets a corpse, at the cost of one cheap probe per acquire. To cut that latency, add the "trust if used within the last N ms" shortcut (most borrows hit it) and lean on a background reaper for the rest. Validate-on-return taxes the hot return path and still can't prevent a conn dying *while idle*. In practice: borrow-check + recently-used shortcut + reaper.

<br>

#### Q4: "What is a connection leak and how do I prevent it?"

A leak is a connection checked out and never returned — usually an early `return` or exception on a path that called manual `release()` after the fault. The pool slowly drains until every `acquire()` times out. Prevention: the **RAII handle** returns the connection in its destructor regardless of how the scope exits. Detection: log/track checkout duration and warn when a connection has been out longer than some threshold (HikariCP's "leak detection threshold").

<br>

#### Q5: "Why must the factory run outside the lock?"

Creating a connection involves a TCP handshake, TLS, and auth — tens to hundreds of milliseconds. If you hold the pool mutex during that, every other thread's `acquire()`/`release()` blocks for the whole duration, serializing the pool and destroying throughput. The pattern: reserve a slot under the lock (`++created_`), drop the lock, do the slow create, then re-take the lock to hand it out. On failure, re-take the lock and roll back `--created_`.

<br>

#### Q6: "How do I size minSize and maxSize?"

`maxSize` is bounded by the *downstream* limit divided by the number of app instances: if Postgres allows 100 connections and you run 10 app pods, each pool's `maxSize` should be ≤ 10 (leave headroom). Counterintuitively, **smaller pools often perform better** — a DB with 4 cores processes ~4 queries truly in parallel; 100 connections just add context-switch and lock contention. HikariCP's rule of thumb: `connections ≈ (core_count * 2) + effective_spindle_count`. `minSize` = enough to absorb baseline traffic without paying creation latency.

<br>

#### Q7: "How does this relate to pgbouncer / connection multiplexing?"

An in-process pool (this exercise) is per-application-instance. **pgbouncer** is an external pool that sits between many app instances and one database, multiplexing thousands of client connections onto a few server connections (transaction-level pooling). Both exist for the same reason — connections are expensive and capped — but the external proxy lets you share the cap across all clients. They compose: a small in-process pool talking to pgbouncer talking to Postgres.

<br>

#### Q8: "What goes wrong if I forget to join the reaper thread in the destructor?"

The reaper keeps running and dereferences `mtx_`, `idle_`, `created_` after the `ConnectionPool` object (and those members) have been destroyed → use-after-free, often a crash or silent corruption. Always: set `stopping_` under the lock, `notify_all()` the shutdown CV, then `join()` the thread *before* member destructors run. The reaper's `wait_for(..., predicate)` ensures it wakes promptly instead of sleeping out the full interval.

<br><br>

---

## Reflection Questions

1. Trace a connection through its lifecycle states. What event triggers each transition, and what is the pool's invariant relating `created`, `idle`, and `in_use`?
2. Why is `wait_until(deadline)` inside a `while` loop the correct way to implement an acquire timeout, rather than `wait_for(remaining)`?
3. Why must connection creation happen outside the pool mutex, and what's the rollback if creation throws?
4. Where are the three places you can run a health check, and what does each one trade off?
5. Why does the reaper keep `minSize` connections warm instead of closing everything idle?
6. What is a connection leak, and how does the RAII handle make it structurally impossible?

---

## Interview Questions

1. "Design a thread-safe connection pool. Walk me through `acquire` with a timeout."
2. "How do you handle the case where all connections are checked out and a new request arrives?"
3. "A connection in the pool was silently killed by the database. How do you avoid handing it to a caller?"
4. "Why might a *smaller* connection pool outperform a larger one?"
5. "Implement the RAII handle that returns a connection to the pool. Why move-only?"
6. "How would you size `maxSize` when you have 20 app instances and a database limit of 200 connections?"
7. "What is a connection leak? How would you detect one in production?"
8. "Explain idle eviction. Why keep a minimum number of connections warm?"
9. "How does an in-process pool relate to an external pooler like pgbouncer?"
10. "How do you cleanly shut down a pool that has a background reaper thread and connections still checked out?"

---

**Next**: Day 44 — Design a KV Store →
