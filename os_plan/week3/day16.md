# Day 16 — Locks: mutex and spinlock

> **Week 3 · Concurrency**
> Reading: OSTEP Chapters 28–29 (Locks, Lock-Based Concurrent Data Structures)

## Why this matters

The mutex is the workhorse of concurrent programming. Spinlocks appear in kernel code, lock-free hot paths, and performance discussions. Knowing when to use which — and what they cost — is core systems knowledge.

## 16.1 The basic mutex

A mutex (mutual-exclusion lock) provides two operations:

```c
pthread_mutex_t lock = PTHREAD_MUTEX_INITIALIZER;

pthread_mutex_lock(&lock);    // blocks if another thread holds it
// critical section
pthread_mutex_unlock(&lock);
```

Semantics:
- At most one thread holds the lock at a time.
- A thread calling `lock` while it's held by another **blocks** — the kernel puts it to sleep.
- When the holder calls `unlock`, the kernel wakes a waiter (one of them; usually FIFO-ish but not guaranteed).

The mutex itself is just a small struct (a few bytes to ~40 bytes depending on platform) with internal state: who holds it, who's waiting, etc.

## 16.2 Mutex implementation: futex

Linux mutexes are implemented with **futexes** (fast userspace mutexes). The clever insight: in the uncontended case (no other thread is waiting), lock and unlock are pure userspace operations — just an atomic CAS on a memory word. No syscall. Microseconds → nanoseconds.

The futex contract:
- **`FUTEX_WAIT(addr, val)`**: if `*addr == val`, sleep. Used by lock() when CAS fails.
- **`FUTEX_WAKE(addr, n)`**: wake up to n threads waiting on `addr`. Used by unlock() when waiters exist.

Pseudocode:

```c
void mutex_lock(int *m) {
    int c = atomic_cas(m, 0, 1);   // 0=free, 1=locked, 2=locked+waiters
    if (c == 0) return;            // got it, no syscall
    while (atomic_xchg(m, 2) != 0) // mark as "locked with waiters"
        futex_wait(m, 2);          // syscall: sleep if still 2
}

void mutex_unlock(int *m) {
    if (atomic_dec(m) != 1)        // was there a waiter?
        atomic_store(m, 0), futex_wake(m, 1);
}
```

So:
- Uncontended lock: one atomic CAS. ~10-30 ns.
- Contended lock: CAS fails, syscall, sleep, eventual wake, retry. ~microseconds to milliseconds.

This is why "mutexes are cheap when uncontended" — there's no kernel involvement at all.

## 16.3 Spinlocks

A spinlock differs in one fundamental way: instead of sleeping when contended, the waiting thread **spins** in a tight loop, repeatedly trying to acquire.

```c
void spin_lock(atomic_int *s) {
    while (atomic_exchange(s, 1) == 1) {
        // already locked, spin
        while (atomic_load(s) == 1) cpu_relax();   // PAUSE on x86
    }
}
void spin_unlock(atomic_int *s) {
    atomic_store(s, 0);
}
```

When to use a spinlock:

- **Critical section is very short** (nanoseconds): the cost of putting to sleep + waking is more than just spinning. A 50-ns spinlock spin is cheap; sleeping costs ~microseconds.
- **In interrupt context** in a kernel: you can't sleep there.
- **No more than one thread per CPU** can spin productively: if you have more spinners than CPUs, they just burn CPU.

When NOT to use:

- **Long critical sections**: spinning wastes CPU. Other threads can't run.
- **User space, especially with M:1 model**: spinning a user thread might block the only kernel thread.
- **Holder might be preempted**: spinning while the holder is descheduled is pure waste.

In Linux user space, you usually want a mutex. The kernel uses spinlocks in many places where it can't sleep.

### Adaptive mutexes

Modern mutexes hybridize: spin briefly first (the holder might release in a microsecond), then if still contended, sleep. glibc's mutex does this. So in practice you rarely need a pure spinlock in userspace.

## 16.4 Lock contention and its costs

When multiple threads frequently want the same lock, you have **contention**. Symptoms:

- Throughput plateaus or drops as you add threads.
- High CPU time in lock/unlock paths (in profiles).
- High `cs` in `vmstat` (context switches as threads sleep and wake).

Costs:

1. **Direct lock overhead**: each lock/unlock cycle is at least an atomic op, often a syscall on contention.
2. **Cache line ping-pong**: the lock word's cache line bounces between CPUs. Each transfer costs ~50–200 cycles plus delays.
3. **Convoy effect**: threads queue up on the lock; the first to get it does work, the rest wait. Throughput is limited to one thread at a time on critical-section work.

Reducing contention:
- **Make critical sections shorter**: do work outside the lock whenever possible.
- **Lock at finer granularity**: per-bucket locks instead of one big map lock.
- **Reader-writer locks**: many readers can share; only writers exclude.
- **Per-CPU / per-thread state**: each thread has its own; merge occasionally.
- **Lock-free algorithms**: atomics + careful design.
- **Batching**: take the lock once for many operations.

## 16.5 Mutex flavors

POSIX defines several mutex types via `pthread_mutexattr_settype`:

| Type | Behavior |
|------|----------|
| `PTHREAD_MUTEX_NORMAL` | Default. Self-locking deadlocks; double-unlock is undefined. |
| `PTHREAD_MUTEX_ERRORCHECK` | Returns errors instead of UB (slower). |
| `PTHREAD_MUTEX_RECURSIVE` | Same thread can re-lock. Tracks count. |
| `PTHREAD_MUTEX_DEFAULT` | Implementation-defined (usually NORMAL). |

Recursive mutexes are sometimes useful (one function holds a lock and calls another that also takes it), but often a sign that lock structure isn't right. Prefer breaking up code so explicit lock/unlock pairs are clear.

Other attributes:
- **`PTHREAD_PROCESS_SHARED`**: lock can be in shared memory and used between processes. Default is `PTHREAD_PROCESS_PRIVATE`.
- **Robust mutexes**: if a thread holding the lock dies (e.g., crashed process), other waiters get `EOWNERDEAD` and can recover. Useful for inter-process locks.
- **Priority inheritance** / **priority ceiling** (Day 5): for RT applications.

## 16.6 Reader-writer locks

When most operations read shared state and writes are rare, a reader-writer lock allows many readers concurrently:

```c
pthread_rwlock_t rwl;
pthread_rwlock_rdlock(&rwl);   // multiple readers can hold simultaneously
// ... read ...
pthread_rwlock_unlock(&rwl);

pthread_rwlock_wrlock(&rwl);   // exclusive
// ... write ...
pthread_rwlock_unlock(&rwl);
```

Pros: read concurrency. Cons:

- Write starvation if readers keep arriving (most implementations have a fairness mode).
- Higher overhead than mutex per acquire (more state to track).
- Cache line still bounces on every lock/unlock — the read concurrency advantage is real only if the critical section is long enough to amortize.

For very read-heavy workloads with extremely rare writers, **RCU** (Read-Copy-Update) is dramatically better — readers have zero overhead, writers do work proportional to updates. RCU is heavily used in the Linux kernel. Userspace RCU exists (`liburcu`); we'll see it again Day 19.

## 16.7 Try-lock and timed lock

Beyond plain `lock`:

- **`pthread_mutex_trylock`**: if the lock is free, take it; otherwise return `EBUSY` immediately. Used to avoid blocking, or to try multiple locks in different orders.
- **`pthread_mutex_timedlock`**: like `lock` but with a timeout.

These are useful for deadlock-avoidance protocols (try, back off, retry).

## 16.8 Common mistakes

1. **Forgetting to unlock on every path** (early return, exception). C++ `std::lock_guard` and Rust's lock guards solve this with RAII.
2. **Locking in different orders in different threads** → deadlock. Day 18.
3. **Holding the lock across blocking I/O**: serializes I/O. Often unnecessary.
4. **Holding the lock across a function call** that itself takes locks: subtle deadlock risk if the inner function someday takes the same lock.
5. **Using a global lock for fine-grained data**: contention. Often you can lock per-bucket.
6. **Not protecting some shared variable** because "it's just one byte": data race regardless of size.

## Hands-on (30 minutes)

1. Compile and benchmark the three counter-fix variants from Day 15:
   - mutex
   - atomic
   - per-thread + final sum

   Use `time ./prog` or `perf stat ./prog`. The atomic should be ~5-10x faster than mutex; per-thread should be ~50x.

2. Profile a contended mutex:
   ```bash
   perf record -g ./contended_program
   perf report
   ```
   Look for `__lll_lock_wait` or futex calls in profile.

3. Use `mutrace` (a glibc-based mutex profiler) if available:
   ```bash
   apt install mutrace 2>/dev/null
   mutrace ./your_program
   ```

4. Examine the futex syscalls a contended mutex makes:
   ```bash
   strace -f -e trace=futex -c ./contended_program
   ```

5. Try a reader-writer lock vs. a mutex on a read-heavy workload:
   ```c
   // version A: pthread_mutex
   // version B: pthread_rwlock with rdlock most of the time
   ```
   Benchmark with 4 reader threads + 1 writer at low write rate.

## Interview questions

### Q1. How is a mutex implemented?

**Answer:** On Linux, mutexes are implemented with **futexes** — fast userspace mutexes. The key insight: in the uncontended case (no waiters), lock and unlock are pure user-space atomic operations on a memory word. No syscall, no kernel involvement.

Concretely:
- The mutex state is one 32-bit integer: 0 = unlocked, 1 = locked uncontended, 2 = locked with waiters.
- `lock`: try a CAS from 0 to 1. If success, you have it. If fail (someone else has it), spin briefly (adaptive mutex), then if still contended, atomic-set to 2 (signaling waiters) and call `futex(FUTEX_WAIT)`. The kernel sleeps the thread on a hash-table queue keyed by the futex address.
- `unlock`: atomic-decrement. If state was 2 (waiters), call `futex(FUTEX_WAKE, 1)` to wake one. Otherwise, just leave at 0.

So uncontended cost is just a CAS — tens of nanoseconds. Contended cost is a syscall plus sleep/wake — microseconds at minimum. This makes mutexes cheap when usage is well-architected (rare contention) and expensive when not.

The futex hash table is global in the kernel; the futex address (or its hash) determines the bucket. Wakeups send threads a signal-like notification; they wake, retry the lock acquire, succeed.

### Q2. When should you use a spinlock instead of a mutex?

**Answer:** A spinlock busy-waits on contention; a mutex sleeps. Use a spinlock when:

1. **The critical section is very short** — nanoseconds to a microsecond. The cost of sleeping (microseconds, plus waking later) outweighs spinning briefly.
2. **In contexts where sleep is illegal** — kernel interrupt handlers, parts of the kernel scheduler itself. Kernels use spinlocks heavily; sleeping in interrupt context is a bug.
3. **Lock holder won't be preempted** — if you can guarantee the holder runs to completion (e.g., disabled preemption in kernel), spinning is brief.

Use a mutex when:
- Critical section is meaningful work (any I/O, any computation longer than a microsecond).
- Number of threads can exceed CPUs (spinners would just waste CPU).
- In user space generally.

In practice, modern user-space mutexes are **adaptive**: they spin briefly (~hundreds of nanoseconds) before sleeping. So a "mutex" already gets the best of both for short critical sections. Pure spinlocks in user space are uncommon outside of specific high-performance contexts.

A common kernel idiom: spinlocks protect short critical sections; longer ones use mutexes (in newer kernel code) or sleeplocks. The kernel has both available.

### Q3. What's a reader-writer lock and when is it worth using?

**Answer:** A reader-writer lock allows multiple readers to hold the lock simultaneously, but writers are exclusive. Conceptual semantics:

- `rdlock`: succeed if no writer holds and no writer is waiting (typically). Multiple readers can hold concurrently.
- `wrlock`: succeed only when no readers and no writer hold.
- `unlock`: release the lock (different code paths for read vs. write release).

Worth using when:
- Reads massively outnumber writes (e.g., 100:1).
- Critical sections are long enough to benefit from parallelism (the per-acquire overhead of a rwlock is higher than a mutex).
- Writers can tolerate being delayed.

Not worth it when:
- Read/write ratio is closer (writes maybe 10%): the overhead exceeds the benefit.
- Critical sections are tiny: per-acquire overhead dominates.
- Cache line still bounces on every acquire/release: you don't get true parallelism for very short reads.

For extremely read-heavy workloads (the lock is barely contended), **RCU** is dramatically better. Readers have effectively zero cost; writers create new versions, switch over, and free old after readers finish. Used in Linux kernel for things like routing tables and the dentry cache.

A subtle pitfall: rwlock can starve writers (continuous reader stream). POSIX says implementations may give writers preference; check your platform. glibc's behavior is configurable via `PTHREAD_RWLOCK_PREFER_WRITER_NONRECURSIVE_NP`.

### Q4. What are the main costs of using locks?

**Answer:** Several costs add up:

1. **Direct overhead per acquire/release**: at minimum an atomic CAS (~10–30 ns uncontended). Under contention, futex syscalls and sleep/wake (microseconds).

2. **Cache line bouncing**: the lock word lives in some cache line. Each acquire/release in different CPUs forces ownership transfer of that line — RFOs (Read-For-Ownership) traffic on the interconnect, ~100+ cycles per transfer. With heavy contention, this dominates.

3. **Serialization (convoy effect)**: only one thread inside the critical section at a time. Throughput is limited to (work done in critical section) per (one thread). Adding more threads doesn't help.

4. **Lock holder preemption**: if the OS deschedules a thread holding a lock, all waiters block longer than necessary.

5. **Indirect overhead**: cache pollution, CPU stall on contention, branch mispredicts in lock paths.

6. **Engineering cost**: deadlock risk, code complexity, debugging difficulty.

Mitigations:
- **Shorter critical sections**: do as much work as possible outside the lock.
- **Finer granularity**: many small locks instead of one big one.
- **Per-CPU / per-thread sharding**: each thread has its own data; merge rarely.
- **Lock-free algorithms**: avoid locks entirely for simple operations.
- **Batching**: amortize lock cost over many operations.

The pattern in scaled systems: start with one big lock, profile, identify hot locks, split or replace as needed. Don't over-engineer up front; locks are simple and correct, that's worth a lot.

## Self-test

1. A mutex's uncontended lock-then-unlock costs about how many nanoseconds? What about contended?
2. Why do kernel spinlocks disable preemption while held?
3. In what scenario would using a reader-writer lock perform *worse* than a regular mutex?
4. Two threads both call `pthread_mutex_lock` on the same uninitialized mutex. What's the behavior?
5. You profile a server and see 30% CPU time in `__lll_unlock_wake`. What does this suggest? What might fix it?
