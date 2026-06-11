# Day 40: Design a Rate Limiter

[← Back to Study Plan](../lld-study-plan.md) | [← Day 39](day39-lru-cache-threadsafe.md)

> **Time**: ~1.5-2 hours
> **Goal**: A rate limiter answers one question — *"is this request allowed right now, or should it be rejected/delayed?"* — and it's a staple of API gateways, login throttling, and any system that must protect a downstream resource from overload. Today you learn the five classic algorithms (**token bucket**, **leaky bucket**, **fixed window**, **sliding window log**, **sliding window counter**), their precise trade-offs (burst tolerance, memory, boundary spikes, smoothing), and you build a **token bucket** with **lazy refill** — `allow()` returns true/false, refilling tokens from elapsed time instead of a background timer.

---
---

# PART 1: THE PROBLEM

---
---

<br>

<h2 style="color: #2980B9;">📘 40.1 What Rate Limiting Is For</h2>

A rate limiter caps how often an action may happen — "100 requests per minute per user," "5 login attempts per 15 minutes per IP," "1000 writes per second to this database." Without one, a buggy client, a traffic spike, or an attacker can overwhelm a service.

The core API is deceptively simple:

```cpp
bool allow();              // is one request permitted right now? (consumes a unit if yes)
```

Everything interesting is *how* you decide. The decision balances competing goals:

| Goal | Tension |
|------|---------|
| **Enforce an average rate** | but allow short bursts (real traffic is bursty) |
| **Smooth load downstream** | but don't reject legitimate spikes |
| **Cheap to compute & store** | but accurate at window boundaries |
| **Fair across clients** | but simple to reason about |

Different algorithms pick different points on these trade-offs. Knowing *which* to use — and *why* — is the whole interview.

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 40.2 Two Dimensions to Reason About</h2>

When comparing algorithms, hold two questions in mind:

1. **Does it allow bursts?** A limit of "60/min" could mean "1 per second, strictly" or "up to 60 at once, then nothing for a minute." These behave very differently downstream.
2. **What does it cost?** Per-client memory and per-request CPU. A limiter tracking millions of users must be cheap per key.

```
   Strict pacing                                Bursty
   (leaky bucket)  ◄───────────────────────►   (token bucket, fixed window)

   Tiny memory                                  More memory
   (token/leaky bucket, fixed window) ◄──────►  (sliding window log)
```

<br><br>

---
---

# PART 2: THE FIVE ALGORITHMS

---
---

<br>

<h2 style="color: #2980B9;">📘 40.3 Fixed Window Counter</h2>

Divide time into fixed windows (e.g., each calendar minute). Keep one counter per window; increment on each request; reject when the counter exceeds the limit. At the window boundary, reset to zero.

```
   limit = 5 per minute
   |── 12:00:00 ──────────── 12:00:59 ──|── 12:01:00 ── ...
        count: 1 2 3 4 5 ✗ ✗            reset → 0
```

```cpp
struct FixedWindow {
    int          limit;
    std::chrono::seconds windowLen;
    int          count = 0;
    std::chrono::steady_clock::time_point windowStart = std::chrono::steady_clock::now();

    bool allow() {
        auto now = std::chrono::steady_clock::now();
        if (now - windowStart >= windowLen) {     // new window → reset
            windowStart = now;
            count = 0;
        }
        if (count < limit) { ++count; return true; }
        return false;
    }
};
```

| Pros | Cons |
|------|------|
| Trivial; O(1) time, O(1) memory per key | **Boundary burst**: up to 2× the limit across a boundary |
| Easy to explain and store (one int + one timestamp) | Coarse — no smoothing within a window |

> **The boundary problem.** With limit 5/min, a client can fire 5 requests at 12:00:59 and 5 more at 12:01:00 — **10 requests in ~1 second**, double the intended rate, because each lands in a different window. The sliding-window algorithms below exist to fix this.

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 40.4 Sliding Window Log</h2>

Store a **timestamp for every request**. To decide, drop timestamps older than the window and count what remains; allow if the count is below the limit.

```
   limit = 5 per 60s, now = 12:01:10
   log: [12:00:12][12:00:40][12:00:55][12:01:02][12:01:08]   (all within last 60s)
        └─ drop anything before 12:00:10 ─┘   count=5 → reject the next
```

```cpp
#include <deque>
struct SlidingWindowLog {
    int          limit;
    std::chrono::seconds window;
    std::deque<std::chrono::steady_clock::time_point> log;

    bool allow() {
        auto now = std::chrono::steady_clock::now();
        while (!log.empty() && now - log.front() >= window) log.pop_front();  // evict old
        if (static_cast<int>(log.size()) < limit) { log.push_back(now); return true; }
        return false;
    }
};
```

| Pros | Cons |
|------|------|
| **Perfectly accurate** — true rolling window, no boundary spike | **O(limit) memory per key** — stores every timestamp |
| Simple to reason about | Expensive at scale (millions of keys × hundreds of timestamps) |

Use it when accuracy matters and limits/keys are small (e.g., strict login throttling). Not for high-cardinality, high-limit cases.

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 40.5 Sliding Window Counter (Approximate)</h2>

A clever compromise: keep just **two** fixed-window counters (current and previous) and **interpolate** based on how far into the current window you are. It approximates the rolling window with O(1) memory.

```
   estimate = current_count + previous_count * (1 - elapsed_fraction_of_current_window)

   limit=10, prev window had 8, current has 3, we're 30% into the current window:
   estimate = 3 + 8 * (1 - 0.30) = 3 + 5.6 = 8.6  → allow (8.6 < 10), else reject
```

```cpp
struct SlidingWindowCounter {
    int          limit;
    std::chrono::milliseconds window;
    int          prevCount = 0, curCount = 0;
    std::chrono::steady_clock::time_point curStart = std::chrono::steady_clock::now();

    bool allow() {
        auto now = std::chrono::steady_clock::now();
        auto elapsed = now - curStart;
        if (elapsed >= window) {
            // roll forward: if we skipped a full window, prev becomes 0
            prevCount = (elapsed < 2 * window) ? curCount : 0;
            curCount  = 0;
            curStart += (elapsed < 2 * window) ? window : elapsed;  // align
            elapsed   = now - curStart;
        }
        double frac = 1.0 - std::chrono::duration<double>(elapsed).count()
                          / std::chrono::duration<double>(window).count();
        double estimate = curCount + prevCount * frac;
        if (estimate < limit) { ++curCount; return true; }
        return false;
    }
};
```

| Pros | Cons |
|------|------|
| O(1) memory (two counters + a timestamp) | Approximate — assumes uniform distribution within the previous window |
| Smooths the fixed-window boundary spike | Slightly over- or under-counts during bursts |
| Used by Cloudflare and many CDNs at scale | More moving parts than fixed window |

This is the pragmatic production default for high-scale API rate limiting: nearly as accurate as the log, but O(1).

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 40.6 Leaky Bucket</h2>

Model a bucket with a hole. Requests pour in; they **leak out at a constant rate**. If the bucket overflows (incoming faster than the leak + capacity), excess requests are rejected. The leak rate **paces** output to a steady stream regardless of input burstiness.

```
        requests in (bursty)
              │ │ │ ││││
              ▼ ▼ ▼ ▼▼▼▼
          ┌───────────────┐
          │  bucket (cap)  │  ← if full, overflow → reject
          └───────┬───────┘
                  │ leaks at fixed rate R
                  ▼
            steady output (smoothed)
```

Two framings:
- **As a queue**: incoming requests wait in a FIFO of bounded size; a worker dequeues at rate R. Overflow when the queue is full.
- **As a meter**: a counter that fills on each request and drains at rate R; reject when adding would exceed capacity.

| Pros | Cons |
|------|------|
| **Smoothest output** — strict constant outflow | **No burst tolerance** — even a brief legitimate spike is delayed/dropped |
| Great for protecting a fragile downstream (fixed-throughput resource) | Queue adds latency; can feel sluggish to clients |

Leaky bucket favors the *downstream's* stability over the *client's* burst needs. Token bucket (next) is the inverse trade-off.

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 40.7 Token Bucket — The Workhorse</h2>

A bucket holds up to `capacity` tokens and is **refilled at `refillRate` tokens per second**. Each request must take one token: if a token is available, allow and consume it; otherwise reject. Because the bucket can hold up to `capacity` tokens, it **tolerates bursts up to `capacity`** while enforcing the average `refillRate` over time.

```
   capacity = 5, refillRate = 1 token/sec

   t=0s:  [●●●●●]  5 tokens (full)
   burst of 5 requests → all allowed, bucket empties:  [     ]
   t=1s:  [●    ]  1 token refilled → 1 more request allowed
   t=2s:  [●    ]  ... steady state: ~1 req/sec, but up to 5 at once if saved up
```

This is the most popular algorithm (NGINX, AWS API Gateway, Stripe, gRPC, Guava `RateLimiter`) because it captures the realistic policy: **"X per second on average, with the flexibility to burst up to a cap."**

| Pros | Cons |
|------|------|
| Allows bursts up to `capacity`, enforces avg `refillRate` | Two parameters to tune (capacity *and* rate) |
| O(1) memory (token count + last-refill timestamp) | Burst right after refill can briefly exceed steady rate (by design) |
| Lazy refill → no background timer needed | — |

<br>

#### Lazy refill — the key implementation idea

Instead of a background thread adding tokens every tick (wasteful, doesn't scale to millions of keys), compute the refill **on demand** from elapsed time:

```
   On each allow():
       now = clock.now()
       elapsed = now - lastRefill
       tokens = min(capacity, tokens + elapsed * refillRate)   // catch up
       lastRefill = now
       if tokens >= 1: tokens -= 1; return true
       else:           return false
```

The bucket "catches up" on tokens lazily at the moment it's queried. No timers, no per-key threads — just arithmetic on a stored timestamp. This is why token bucket scales to millions of keys (each is two numbers).

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 40.8 Side-by-Side</h2>

| Algorithm | Burst tolerance | Output smoothing | Memory / key | Boundary spike | Typical use |
|-----------|-----------------|------------------|--------------|----------------|-------------|
| **Fixed window** | Up to limit (per window) | None | O(1) | **Yes (2×)** | Simple quotas, cheap |
| **Sliding window log** | Exact | None | **O(limit)** | No | Strict, low-volume (login) |
| **Sliding window counter** | ~limit | Some | O(1) | Minimal | High-scale APIs (Cloudflare) |
| **Leaky bucket** | **None** | **Strict constant** | O(1)–O(queue) | No | Protect fragile downstream |
| **Token bucket** | **Up to capacity** | Average rate | O(1) | No | General API limiting (most common) |

The two you'll reach for most: **token bucket** (allow bursts, enforce average) and **sliding window counter** (accurate at scale, no boundary spike). Leaky bucket when the downstream demands a strictly steady feed.

<br><br>

---
---

# PART 3: BUILD EXERCISE

---
---

<br>

<h2 style="color: #2980B9;">📘 40.9 Exercise: Token Bucket with Lazy Refill</h2>

Build a token bucket limiter with `allow()` (consume one token) and `allow(n)` (consume n). Use lazy refill from a timestamp. Make it injectable-clock-friendly so the tests are deterministic (no `sleep`). Then add a thread-safe wrapper.

<br>

#### Skeleton

```cpp
// token_bucket.h
#pragma once
#include <algorithm>
#include <chrono>
#include <mutex>

// A clock abstraction so tests can advance time deterministically.
struct SteadyClock {
    using time_point = std::chrono::steady_clock::time_point;
    static time_point now() { return std::chrono::steady_clock::now(); }
};

template <typename Clock = SteadyClock>
class TokenBucket {
    double                    m_capacity;      // max tokens
    double                    m_refillPerSec;  // tokens added per second
    double                    m_tokens;        // current tokens (fractional ok)
    typename Clock::time_point m_last;         // last refill time

    void refill(typename Clock::time_point now) {
        double elapsed = std::chrono::duration<double>(now - m_last).count();
        if (elapsed <= 0) return;
        m_tokens = std::min(m_capacity, m_tokens + elapsed * m_refillPerSec);
        m_last   = now;
    }

public:
    TokenBucket(double capacity, double refillPerSec)
        : m_capacity(capacity), m_refillPerSec(refillPerSec),
          m_tokens(capacity), m_last(Clock::now()) {}

    // Try to consume n tokens. Returns true (and consumes) if available.
    bool allow(double n = 1.0) {
        auto now = Clock::now();
        refill(now);
        if (m_tokens >= n) { m_tokens -= n; return true; }
        return false;
    }

    double tokens() {
        refill(Clock::now());
        return m_tokens;
    }
};

// ── Thread-safe wrapper ─────────────────────────────────
template <typename Clock = SteadyClock>
class ThreadSafeTokenBucket {
    TokenBucket<Clock> m_bucket;
    std::mutex         m_mtx;
public:
    ThreadSafeTokenBucket(double cap, double rate) : m_bucket(cap, rate) {}
    bool allow(double n = 1.0) {
        std::lock_guard<std::mutex> lk(m_mtx);
        return m_bucket.allow(n);
    }
};
```

<br>

#### A fake clock for deterministic tests

```cpp
// A controllable clock — tests advance time without sleeping.
struct FakeClock {
    using time_point = std::chrono::steady_clock::time_point;
    static inline time_point current = std::chrono::steady_clock::now();
    static time_point now() { return current; }
    static void advance(std::chrono::milliseconds d) { current += d; }
};
```

<br>

#### Test driver

```cpp
// main.cpp
#include "token_bucket.h"
#include <atomic>
#include <cassert>
#include <iostream>
#include <thread>
#include <vector>

struct FakeClock {
    using time_point = std::chrono::steady_clock::time_point;
    static inline time_point current = std::chrono::steady_clock::now();
    static time_point now() { return current; }
    static void advance(std::chrono::milliseconds d) { current += d; }
};

int main() {
    using namespace std::chrono;

    // 1. Initial burst up to capacity, then exhausted
    {
        TokenBucket<FakeClock> tb(/*cap=*/5, /*rate=*/1);   // 5 tokens, +1/sec
        for (int i = 0; i < 5; ++i) assert(tb.allow());     // burst of 5 OK
        assert(!tb.allow());                                 // 6th rejected (empty)
        std::cout << "[1] initial burst OK\n";
    }

    // 2. Lazy refill from elapsed time
    {
        TokenBucket<FakeClock> tb(5, 1);
        for (int i = 0; i < 5; ++i) tb.allow();              // drain
        assert(!tb.allow());
        FakeClock::advance(seconds(2));                      // +2s → +2 tokens
        assert(tb.allow());                                  // token available
        assert(tb.allow());                                  // second one
        assert(!tb.allow());                                 // only 2 refilled
        std::cout << "[2] lazy refill OK\n";
    }

    // 3. Refill caps at capacity (no overflow accumulation)
    {
        TokenBucket<FakeClock> tb(5, 1);
        FakeClock::advance(seconds(100));                    // would add 100, but caps at 5
        for (int i = 0; i < 5; ++i) assert(tb.allow());
        assert(!tb.allow());                                 // never exceeded capacity
        std::cout << "[3] capacity cap OK\n";
    }

    // 4. Fractional / multi-token consume
    {
        TokenBucket<FakeClock> tb(10, 5);
        assert(tb.allow(7));                                 // take 7 of 10
        assert(!tb.allow(5));                                // only 3 left
        assert(tb.allow(3));                                 // exactly 3
        std::cout << "[4] multi-token consume OK\n";
    }

    // 5. Average rate enforced over time (steady state)
    {
        TokenBucket<FakeClock> tb(1, 10);                    // cap 1, 10/sec → ~1 per 100ms
        assert(tb.allow());                                  // initial token
        assert(!tb.allow());                                 // empty
        FakeClock::advance(milliseconds(100));               // +1 token
        assert(tb.allow());
        assert(!tb.allow());
        std::cout << "[5] steady-state rate OK\n";
    }

    // 6. Thread-safe wrapper: never over-issues under concurrency
    {
        // Real clock here; cap large, rate 0 so NO refill happens during the test.
        ThreadSafeTokenBucket<> tb(/*cap=*/1000, /*rate=*/0);
        std::atomic<int> granted{0};
        std::vector<std::thread> ts;
        for (int t = 0; t < 8; ++t)
            ts.emplace_back([&]{
                for (int i = 0; i < 1000; ++i)
                    if (tb.allow()) granted.fetch_add(1, std::memory_order_relaxed);
            });
        for (auto& t : ts) t.join();
        // 8000 attempts, only 1000 tokens, no refill → exactly 1000 granted.
        std::cout << "[6] concurrent granted=" << granted.load() << " (expect 1000)\n";
        assert(granted.load() == 1000);
    }

    std::cout << "All assertions passed.\n";
    return 0;
}
```

<br>

#### Compile and run

```bash
# Correctness + races:
g++ -std=c++17 -Wall -Wextra -Wpedantic -fsanitize=thread \
    -pthread -o day40_tsan main.cpp && ./day40_tsan

# Or with the address sanitizer:
g++ -std=c++17 -Wall -Wextra -Wpedantic -fsanitize=address \
    -pthread -o day40 main.cpp && ./day40
```

<br>

#### Expected output

```
[1] initial burst OK
[2] lazy refill OK
[3] capacity cap OK
[4] multi-token consume OK
[5] steady-state rate OK
[6] concurrent granted=1000 (expect 1000)
All assertions passed.
```

Test 6 is the crucial concurrency check: 8 threads make 8000 attempts against a 1000-token bucket with zero refill, and **exactly 1000** succeed — proving the mutex prevents two threads from consuming the same token. ThreadSanitizer reports no races.

<br>

#### Bonus Challenges

1. **`retryAfter()`** — when `allow()` rejects, return how long until enough tokens accumulate: `(n - tokens) / refillPerSec`. Clients use it for a `Retry-After` header / backoff.

2. **Per-key limiter registry** — wrap a `unordered_map<Key, TokenBucket>` so each user/IP gets its own bucket. Combine with Day 38's LRU to evict idle keys (don't leak memory on a billion one-off IPs).

3. **Leaky bucket comparison** — implement leaky bucket (as a meter) with the same interface and feed both a bursty trace. Plot output: token bucket lets the burst through; leaky bucket paces it.

4. **Sliding window counter** — implement §40.5 and demonstrate that fixed-window allows a 2× boundary burst while sliding window counter does not.

5. **Distributed token bucket** — sketch how you'd move this to Redis (atomic Lua script doing the refill-and-consume) so a cluster of API servers shares one limit per key. What consistency issues arise?

6. **Lock-free token bucket** — replace the mutex with a CAS loop on an atomic token count (pack tokens + timestamp, or use a `compare_exchange` retry). Benchmark against the locked version and check correctness with TSan.

<br><br>

---
---

# PART 4: DEEP DIVE — COMMON QUESTIONS

---
---

<br>

<h2 style="color: #2980B9;">📘 40.10 Q&A</h2>

<br>

#### Q1: "Token bucket vs leaky bucket — what's the real difference?"

Token bucket enforces an **average** rate but *allows bursts* up to the bucket capacity (tokens accumulate while idle, then can be spent at once). Leaky bucket enforces a **strict constant output** rate with *no burst tolerance* — it smooths bursty input into a steady stream. Token bucket favors client flexibility; leaky bucket favors downstream stability. Token bucket is the more common API-limiting choice; leaky bucket protects fixed-throughput resources.

<br>

#### Q2: "What is lazy refill and why use it instead of a timer?"

Lazy refill computes tokens on demand from elapsed time (`tokens += elapsed × rate`, capped at capacity) at the moment `allow()` is called, instead of a background thread topping up every tick. It needs no timers or per-key threads — each bucket is just a token count plus a last-refill timestamp — so it scales to millions of keys with O(1) memory and no idle CPU.

<br>

#### Q3: "What's the boundary problem with fixed windows?"

Two adjacent windows each permit up to `limit` requests, so a client firing `limit` requests at the end of one window and `limit` at the start of the next sends `2 × limit` in a very short span — double the intended rate. Sliding window log (exact) and sliding window counter (approximate, O(1)) eliminate this by considering a rolling window.

<br>

#### Q4: "When would you use sliding window log despite its memory cost?"

When you need **exact** enforcement and the volume is low — e.g., "5 failed logins per 15 minutes per account." The O(limit) per-key memory is fine when limits and key counts are small, and exactness matters for security. For high-cardinality, high-limit API traffic, switch to the sliding window counter.

<br>

#### Q5: "How do you make a token bucket thread-safe efficiently?"

A mutex around `refill + consume` is correct and usually fast (the critical section is a few arithmetic ops). For extreme contention, use a CAS loop on an atomic representation of the token count (lock-free), retrying when another thread wins the race. But measure first — the mutex version is rarely the bottleneck, and the I/O the limiter protects is far slower.

<br>

#### Q6: "How do you rate-limit per user without leaking memory?"

Keep a map of key → bucket, but bound it: evict idle buckets with an LRU (Day 38) or TTL so a flood of one-off keys (e.g., random IPs) can't grow the map unbounded. Re-creating a bucket for a returning idle key is harmless (it would have refilled to full anyway).

<br>

#### Q7: "How does rate limiting work across a fleet of servers?"

Local per-server limiters are simplest but let each server permit the full limit (N servers → N× the intended global rate). For a true global limit, centralize the counter — typically Redis with an atomic Lua script doing refill-and-consume — accepting added latency and a dependency. A hybrid: local limiters sized to `globalLimit / N`, periodically reconciled.

<br>

#### Q8: "What should happen on rejection — drop, queue, or delay?"

Three policies: **reject** immediately (return 429 Too Many Requests + `Retry-After`) — best for APIs; **queue/delay** (leaky-bucket style) — smooths load but adds latency and risks unbounded queues; **shed** selectively (drop low-priority, keep high-priority). The right choice depends on whether the client can retry and whether latency or loss is worse for your use case.

<br><br>

---

## Reflection Questions

1. What is the precise behavioral difference between token bucket and leaky bucket?
2. Why does lazy refill scale to millions of keys where a timer-based refill does not?
3. Explain the fixed-window boundary problem and how each sliding-window variant addresses it.
4. When is the O(limit) memory of a sliding window log justified?
5. How do you bound the memory of a per-key rate limiter?
6. What changes when a rate limit must be enforced globally across many servers?

---

## Interview Questions

1. "Design a rate limiter. Walk through the algorithm choices and trade-offs."
2. "Implement a token bucket with lazy (timestamp-based) refill."
3. "Token bucket vs leaky bucket — when would you choose each?"
4. "What's the boundary problem with fixed-window counters? How do you fix it?"
5. "Compare sliding window log and sliding window counter on memory and accuracy."
6. "How would you rate-limit per user/IP without leaking memory?"
7. "How do you enforce a single global rate limit across a fleet of servers?"
8. "Make your token bucket thread-safe. How would you do it lock-free?"
9. "On rejection, should you drop, delay, or queue the request? Defend your answer."
10. "How would you return a `Retry-After` hint to the client?"

---

**Next**: Day 41 — Design a Timer / Scheduler →
