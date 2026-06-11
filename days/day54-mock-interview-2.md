# Day 54: Mock Interview #2

[← Back to Study Plan](../lld-study-plan.md) | [← Day 53](day53-mock-interview-1.md)

> **Time**: ~1.5-2 hours
> **Goal**: Second timed mock. Today's set: **Rate Limiter** (Day 40, `day40-rate-limiter.md`), **SharedPtr** (Day 3, `day03-shared-ptr.md`), and **Timer** (Day 41, `day41-timer.md`). The worked example is the **Rate Limiter**. The emphasis this round is **articulating trade-offs aloud** — these three problems each have several defensible designs (token bucket vs sliding window; intrusive vs control-block refcount; one-thread-per-timer vs a single timer wheel), and the interviewer's real question is always *"why this one?"* You pass by *narrating the comparison*, not by silently picking.

---
---

# PART 1: HOW TO RUN A TIMED MOCK (TRADE-OFF EMPHASIS)

---
---

<br>

<h2 style="color: #2980B9;">📘 54.1 Same Clock, Sharper Narration</h2>

The 60-minute structure from Day 53 still holds:

```
 0:00 ─ 0:05   Clarify
 0:05 ─ 0:20   Design on paper  ◄── today: list ≥2 designs and pick, out loud
 0:20 ─ 0:55   Implement
 0:55 ─ 1:00   Test trace + follow-ups
```

The one change today: in the design phase, **explicitly enumerate the candidate designs before committing**. The script is always:

> "There are a few ways to do this — A, B, C. A is simplest but [weakness]. B is more accurate but [cost]. Given the constraints we agreed on — [constraint] — I'll go with B because [reason]. If [constraint changed], I'd switch to A."

That sentence pattern is worth memorizing. It demonstrates breadth (you know the options), judgment (you can choose), and humility (the choice is conditional on requirements).

<br>

<h2 style="color: #2980B9;">📘 54.2 The Trade-off Vocabulary</h2>

Interviewers listen for these axes. Touch the relevant ones for every decision:

| Axis | The question to answer aloud |
|------|------------------------------|
| **Time complexity** | Per-operation big-O, amortized vs worst case |
| **Space complexity** | Memory per entry; fixed vs growing |
| **Accuracy** | Exact vs approximate (matters for rate limiting!) |
| **Concurrency** | Lock granularity, contention, lock-free feasibility |
| **Latency vs throughput** | Tail latency vs aggregate rate |
| **Simplicity / maintainability** | Can a teammate understand it? |
| **Failure mode** | What breaks under overload, clock skew, restart? |

A senior answer rarely says "X is better." It says "X is better *on accuracy* but worse *on memory*; here we care more about accuracy, so X."

<br>

<h2 style="color: #2980B9;">📘 54.3 Reading the Interviewer's Steering</h2>

When the interviewer asks "what about edge case Y?" they are *not* saying you're wrong — they're sampling your robustness. Acknowledge, address, and tie back to a trade-off:

> "Good point — clock going backwards (NTP adjustment) would break a timestamp-based window. I'd use a steady (monotonic) clock instead of wall-clock, accepting that I can't reason about absolute time, only elapsed time."

<br><br>

---
---

# PART 2: WORKED EXAMPLE — RATE LIMITER

---
---

<br>

<h2 style="color: #2980B9;">📘 54.4 Step 1 — Clarify (≈5 min)</h2>

Prompt: *"Design a rate limiter: allow at most R requests per second per client."*

| Question | Assumed answer |
|----------|----------------|
| Per-client or global? | Per-client, keyed by client ID. |
| Hard cap or allow bursts? | Allow short bursts up to a bucket capacity. |
| Exact or approximate is fine? | Approximate within a small margin is acceptable. |
| Single process or distributed? | Single process for v1; mention Redis for distributed as a follow-up. |
| Multi-threaded callers? | Yes — must be thread-safe. |

<br>

<h2 style="color: #2980B9;">📘 54.5 Step 2 — Enumerate Designs, Then Choose (≈10 min)</h2>

This is the heart of today's mock. Lay out the four classic algorithms *and their trade-offs* before picking:

| Algorithm | Idea | Pros | Cons |
|-----------|------|------|------|
| **Fixed window** | Count requests in each 1s bucket; reset each second | Trivial, O(1), tiny memory | Boundary burst: 2× rate possible across the window edge |
| **Sliding window log** | Store timestamp of every request; count those in last 1s | Exact | O(R) memory per client; expensive at high R |
| **Sliding window counter** | Weighted blend of current + previous fixed window | Good accuracy, O(1) | Approximate; assumes uniform distribution |
| **Token bucket** | Tokens refill at rate R, each request spends one; bucket caps burst | O(1), allows controlled bursts, smooth | Burst size is a tuning knob |

The narration:

> "Fixed window is simplest but allows a 2× burst at the boundary — bad for a hard SLA. Sliding-window-log is exact but O(R) memory per client, which won't scale to millions of clients. Token bucket is O(1) in time *and* space per client, allows a controlled burst, and refills smoothly — it matches 'allow short bursts up to a capacity.' I'll implement **token bucket**, and mention sliding-window-counter as the alternative if bursts were unacceptable."

```
            Token Bucket (capacity C, refill rate R/sec)

   refill ──► ┌───────────────┐
   R/sec      │ ● ● ● ● ●      │  tokens (≤ C)
              └───────────────┘
                     │ allow() spends 1 token if available
                     ▼
              request admitted / rejected
   tokens(now) = min(C, tokens(last) + (now - last) * R)
```

<br>

<h2 style="color: #2980B9;">📘 54.6 Step 3 — API Sketch & Reference Implementation (≈35 min)</h2>

**API:**

```cpp
class RateLimiter {
public:
    RateLimiter(double rate_per_sec, double burst_capacity);
    bool allow(int n = 1);          // try to consume n tokens; false if not enough
};
```

The key implementation insight to *say aloud*: **lazy refill**. We don't run a background thread adding tokens; we compute how many tokens *would have* accrued since the last call, based on elapsed time on a **steady clock**. O(1), no timer thread.

```cpp
#include <algorithm>
#include <chrono>
#include <mutex>

class RateLimiter {
    using Clock = std::chrono::steady_clock;   // monotonic — immune to wall-clock jumps

    const double rate_;        // tokens per second
    const double capacity_;    // max burst
    double       tokens_;      // current tokens
    Clock::time_point last_;   // last refill timestamp
    std::mutex   mtx_;

public:
    RateLimiter(double rate_per_sec, double burst_capacity)
        : rate_(rate_per_sec),
          capacity_(burst_capacity),
          tokens_(burst_capacity),
          last_(Clock::now()) {}

    bool allow(int n = 1) {
        std::lock_guard<std::mutex> lk(mtx_);
        refill();
        if (tokens_ >= n) {
            tokens_ -= n;
            return true;
        }
        return false;
    }

private:
    void refill() {
        auto now = Clock::now();
        double elapsed =
            std::chrono::duration<double>(now - last_).count();
        tokens_ = std::min(capacity_, tokens_ + elapsed * rate_);
        last_   = now;
    }
};
```

For the **per-client** dimension, wrap a map (mention sharding for contention):

```cpp
class PerClientLimiter {
    double rate_, cap_;
    std::unordered_map<std::string, RateLimiter> limiters_;
    std::mutex map_mtx_;
public:
    PerClientLimiter(double r, double c) : rate_(r), cap_(c) {}
    bool allow(const std::string& client) {
        std::lock_guard<std::mutex> lk(map_mtx_);   // shard this in production
        auto it = limiters_.find(client);
        if (it == limiters_.end())
            it = limiters_.emplace(client, RateLimiter(rate_, cap_)).first;
        return it->second.allow();
    }
};
```

Things to verbalize while coding:
1. *"Steady clock, not system clock — an NTP correction or DST jump must not grant or deny tokens."*
2. *"Lazy refill means zero background threads and O(1) per call — better than a refill timer per client."*
3. *"The per-client map needs its own lock; under contention I'd shard by `hash(client) % N`, and evict idle clients with a TTL so the map doesn't grow unbounded."*

<br>

<h2 style="color: #2980B9;">📘 54.7 Step 4 — Trade-offs to Articulate</h2>

| Decision | Chosen | Alternative | Switch when |
|----------|--------|-------------|-------------|
| Algorithm | Token bucket | Sliding-window counter | Bursts unacceptable; need smoother cap |
| Refill | Lazy (compute on access) | Background refill thread | Need pre-granted tokens with no access |
| Clock | `steady_clock` | `system_clock` | Never for limiting — wall clock can jump |
| Per-client storage | One map + lock | Sharded maps; LRU/TTL eviction | Many clients or high contention |
| Distributed | Out of scope (single process) | Redis INCR/Lua token bucket | Multiple app servers share the limit |

<br><br>

---
---

# PART 3: SELF-EVALUATION RUBRIC

---
---

<br>

<h2 style="color: #2980B9;">📘 54.8 Score Yourself (trade-off weighting)</h2>

Same 0–4 scale as Day 53, but this round **double-weight "communication"** — it's today's focus.

| Axis | What "strong (3–4)" looks like this round |
|------|-------------------------------------------|
| **Correctness** | Lazy refill math right; steady clock; no token over-grant |
| **API design** | `allow(n)` returns bool; per-client wrapper clean |
| **Concurrency safety** | Lock around refill+consume is atomic; map lock noted/sharded |
| **Communication (×2)** | Enumerated ≥2 algorithms with trade-offs *before* coding; tied each choice to a stated constraint |
| **Complexity analysis** | Stated O(1) time/space per call; memory grows with #clients (and the eviction fix) |

**Did you say the §54.1 sentence pattern at least three times?** If not, that's the gap — knowing the answer silently scores the same as not knowing it.

<br>

<h2 style="color: #2980B9;">📘 54.9 Common Failure Modes This Round</h2>

- **Picking token bucket without naming the alternatives.** Right answer, missing signal.
- **Using `system_clock`.** A classic correctness bug under clock skew.
- **Forgetting the per-client map grows forever.** Always mention eviction/TTL.
- **Holding one global lock for all clients.** Mention sharding even if you don't code it.

<br><br>

---
---

# PART 4: FOLLOW-UP "WHAT-IF" QUESTIONS

---
---

<br>

<h2 style="color: #2980B9;">📘 54.10 Rate Limiter Follow-ups</h2>

#### "Make it distributed across N app servers."

The token state must be shared. Use Redis: a Lua script that atomically reads tokens, refills based on stored timestamp, and decrements — keeping the lazy-refill logic but server-side and atomic. Trade-off: a network round-trip per request (latency) and Redis as a dependency/SPOF. Alternative: each server gets `R/N` local quota (no coordination, but wastes capacity if load is skewed).

#### "Boundary burst in fixed window — explain and fix."

If R=100/s, a client can send 100 at t=0.999s and 100 at t=1.001s — 200 in ~2ms across the reset. Fix with the **sliding-window-counter**: weight the previous window's count by the fraction of it still in view. Token bucket sidesteps this entirely by capping burst to bucket capacity.

#### "How would you return *when* to retry (Retry-After)?"

Compute the time until enough tokens accrue: `(n - tokens_) / rate_` seconds. Return it so the client can back off precisely instead of polling.

<br>

<h2 style="color: #2980B9;">📘 54.11 SharedPtr Follow-ups (problem 2)</h2>

Design recap: a control block holding `strong_count` + `weak_count`, separate from the managed object; copy bumps strong, destruction decrements and frees the object at strong==0 and the control block at weak==0. `make_shared` fuses object + control block into one allocation. (See Day 3, `day03-shared-ptr.md`.)

#### "Make the refcount thread-safe — and is the *pointee* then safe?"

The refcount must be `std::atomic` (relaxed for increment, acq_rel for the decrement that may trigger deletion). But this only makes *control-block* operations safe — concurrent reads/writes to the *pointed-to object* are still your responsibility. State this distinction clearly; it's a frequent trap.

#### "`make_shared` vs `shared_ptr<T>(new T)` — trade-offs?"

`make_shared` = one allocation (object + control block fused), better locality, exception-safe. Downside: the object's memory isn't freed until the *last weak_ptr* dies (weak refs pin the whole fused block). Two-arg form = two allocations but the object memory is reclaimed at strong==0 independent of weak refs.

#### "Why `weak_ptr`, and how does `lock()` work atomically?"

`weak_ptr` observes without owning — breaks cycles (Day 3's parent/child example) and caches. `lock()` atomically checks `strong_count != 0` and, if so, increments it, returning a `shared_ptr` (else empty). The atomicity is essential: a naive "if alive then make shared" races with the last owner's destruction.

<br>

<h2 style="color: #2980B9;">📘 54.12 Timer Follow-ups (problem 3)</h2>

Design recap: schedule callbacks to fire after a delay / at an interval. Naive design = one thread per timer (doesn't scale). Better = a single worker thread with a **min-heap (priority queue) keyed by fire time**, sleeping until the earliest deadline on a `condition_variable` (so new earlier timers wake it). (See Day 41, `day41-timer.md`.)

#### "One thread per timer vs a single timer thread — trade-offs?"

Thread-per-timer is trivial but costs a thread (~MB stack + scheduler load) each, and dies at thousands of timers. Single thread + min-heap is O(log n) to add/cancel and one thread total. For *millions* of timers with coarse resolution, a **hierarchical timing wheel** gives O(1) insert/expire (how Linux kernel timers and Kafka's purgatory work).

#### "How do you cancel a scheduled timer?"

Hand back a token/ID on schedule; cancellation marks the entry dead (tombstone) — you can't cheaply remove from the middle of a binary heap, so skip dead entries when they reach the top. Or use a structure with O(log n) erase. Mention the tombstone approach as the simple correct one.

#### "A new timer is scheduled earlier than the one we're sleeping on. Now what?"

The worker sleeps via `cv.wait_until(earliest_deadline)`. Scheduling notifies the cv, which re-evaluates the heap top and sleeps until the *new* earliest deadline. Without the notify, the new earlier timer would fire late. This is the key correctness point — say it explicitly.

<br><br>

---

## Reflection Questions

1. Did you enumerate ≥2 designs before committing on every problem? Where did you skip it and just pick?
2. Why is a *steady* (monotonic) clock mandatory for a rate limiter, and what bug appears with a wall clock?
3. Atomic refcounts make `shared_ptr` thread-safe in what sense — and explicitly *not* in what sense?
4. For the timer, why must scheduling a new timer notify the sleeping worker? What fails otherwise?
5. Which trade-off did you state least convincingly? What vocabulary (from §54.2) were you missing?
6. Across all three problems, where would *sharding* or *eviction* prevent unbounded growth/contention?

---

## Interview Questions

1. "Design a rate limiter. Compare fixed window, sliding window, and token bucket."
2. "Why use a steady clock for rate limiting? What breaks with system_clock?"
3. "Explain the fixed-window boundary burst and two ways to fix it."
4. "Extend the rate limiter to be distributed across many servers."
5. "Implement `shared_ptr`. Where does the refcount live and why a separate control block?"
6. "`make_shared` vs `shared_ptr(new T)` — trade-offs in allocation and weak_ptr lifetime."
7. "How does `weak_ptr::lock()` work atomically, and why is the atomicity required?"
8. "In what sense is an atomic-refcount `shared_ptr` thread-safe, and in what sense is it not?"
9. "Design a timer service. Why a min-heap + single thread over one-thread-per-timer?"
10. "How do you cancel a timer in a binary-heap-based scheduler? Why tombstones?"

---

**Next**: Day 55 — Mock Interview #3 →
