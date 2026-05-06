# Day 17 — Condition variables and semaphores

> **Week 3 · Concurrency**
> Reading: OSTEP Chapter 30 (Condition Variables); Chapter 31 (Semaphores)

## Why this matters

Mutexes provide mutual exclusion. But often you need to wait for a condition: "wait until the queue is non-empty," "wait until at least 5 workers have registered." Condition variables and semaphores are the tools for these wait-for-event patterns. Producer-consumer queues are the canonical example and a frequent interview question.

## 17.1 Condition variables

A condition variable lets a thread sleep until another thread signals that some condition might be true. Always paired with a mutex.

The pattern:

```c
pthread_mutex_t mu = PTHREAD_MUTEX_INITIALIZER;
pthread_cond_t cv = PTHREAD_COND_INITIALIZER;
int ready = 0;

// Waiter:
pthread_mutex_lock(&mu);
while (!ready)
    pthread_cond_wait(&cv, &mu);    // atomically: release mu, sleep
                                     // on wake: re-acquire mu
// ... use the now-ready data ...
pthread_mutex_unlock(&mu);

// Signaler:
pthread_mutex_lock(&mu);
ready = 1;
pthread_cond_signal(&cv);    // wake one waiter
pthread_mutex_unlock(&mu);
```

Key contract:
- `cond_wait` atomically unlocks the mutex and sleeps. When woken, it re-acquires the mutex before returning. Without atomicity, you'd race: check condition, decide to sleep, but signaler signals before you actually sleep — you sleep forever.
- The check is **always** in a `while` loop, never `if`. Spurious wakeups happen (the OS may wake you even without a signal), and even non-spurious wakes might find the condition false (another thread got there first).

## 17.2 Why a while loop?

Two reasons:

1. **Spurious wakeups**: POSIX allows `cond_wait` to return without a signal. Implementations sometimes do this to simplify internals. If you don't recheck, you proceed with the condition false — bug.

2. **Waking multiple waiters**: with `cond_broadcast`, all waiters wake. Each re-acquires the mutex one at a time. The first might consume the resource that all of them are waiting for — others should re-check and possibly sleep again.

```c
while (!condition)
    pthread_cond_wait(&cv, &mu);
```

This pattern is non-negotiable. Using `if` is a real bug, even if it appears to work most of the time.

## 17.3 Signal vs. broadcast

Two ways to wake waiters:

- **`pthread_cond_signal`**: wake one waiter (implementation choice which one).
- **`pthread_cond_broadcast`**: wake all waiters.

When to use which:

- **Signal**: when only one waiter can usefully proceed (e.g., one item produced — only one consumer can take it).
- **Broadcast**: when all waiters might proceed (e.g., a state change like "shutting down" affects everyone), or when the condition is complex and you can't tell who can proceed.

Broadcast is the safer default — using `signal` requires reasoning about which waiter is the "right" one, and getting it wrong leads to thundering-herd bugs (waking too many) or lost wakeups (waking none). Performance-wise, signal can be cheaper if applicable.

## 17.4 Producer/consumer queue

The canonical use of a condition variable. A bounded queue with multiple producers and consumers:

```c
#define N 16
int buffer[N];
int count = 0, head = 0, tail = 0;
pthread_mutex_t mu = PTHREAD_MUTEX_INITIALIZER;
pthread_cond_t not_empty = PTHREAD_COND_INITIALIZER;
pthread_cond_t not_full  = PTHREAD_COND_INITIALIZER;

void produce(int item) {
    pthread_mutex_lock(&mu);
    while (count == N)
        pthread_cond_wait(&not_full, &mu);
    buffer[tail] = item;
    tail = (tail + 1) % N;
    count++;
    pthread_cond_signal(&not_empty);
    pthread_mutex_unlock(&mu);
}

int consume(void) {
    pthread_mutex_lock(&mu);
    while (count == 0)
        pthread_cond_wait(&not_empty, &mu);
    int item = buffer[head];
    head = (head + 1) % N;
    count--;
    pthread_cond_signal(&not_full);
    pthread_mutex_unlock(&mu);
    return item;
}
```

```mermaid
sequenceDiagram
    participant P as Producer
    participant Q as Queue (mu, count)
    participant C as Consumer

    Note over Q: count = 0
    C->>Q: lock(mu); count==0 → wait(not_empty, mu); sleep
    P->>Q: lock(mu); count++; signal(not_empty); unlock(mu)
    Note over C: woken by signal; re-acquires mu
    C->>Q: count > 0 → take item; signal(not_full); unlock(mu)
```

**Two condition variables, one mutex**: producers wait on `not_full`, consumers on `not_empty`. The single mutex protects all queue state. Each side signals the other after modification.

## 17.5 Semaphores

A semaphore is a non-negative integer with two atomic operations:

- **`sem_wait` (P, decrement)**: if value > 0, decrement and return. Else, block until value > 0.
- **`sem_post` (V, increment)**: increment and wake one waiter if any.

POSIX:
```c
#include <semaphore.h>
sem_t s;
sem_init(&s, 0, 1);    // pshared=0 (process-private), initial value 1

sem_wait(&s);    // try to decrement
// ... critical section ...
sem_post(&s);    // increment, wake one waiter
```

A semaphore initialized to 1 acts like a mutex (binary semaphore). Initialized to N, it allows up to N concurrent holders — useful for resource pools (e.g., "at most 10 simultaneous downloads"):

```c
sem_t download_slots;
sem_init(&download_slots, 0, 10);

void download() {
    sem_wait(&download_slots);
    // do work; up to 10 threads concurrently
    sem_post(&download_slots);
}
```

Semaphores can also do producer-consumer without a separate mutex (counting semaphores):

```c
sem_t empty_slots, filled_slots;
sem_init(&empty_slots, 0, N);
sem_init(&filled_slots, 0, 0);
pthread_mutex_t mu = PTHREAD_MUTEX_INITIALIZER;   // only for buffer access

void produce(int item) {
    sem_wait(&empty_slots);    // wait for empty slot
    pthread_mutex_lock(&mu);
    buffer[tail] = item; tail = (tail+1) % N;
    pthread_mutex_unlock(&mu);
    sem_post(&filled_slots);   // signal a filled slot
}
```

## 17.6 Mutex+CV vs. semaphore — which to choose

Both can express the same patterns. Convention:

- **Mutex+CV**: more general; the condition can be arbitrary (check any predicate). Better for complex synchronization.
- **Semaphore**: more concise when the synchronization is "count of available resources." Producer-consumer slot counts, resource pools.

In modern code, mutex + condition variable is the more common idiom. C++ `std::condition_variable` and Java's `Object.wait/notify` follow this pattern.

Semaphores have a subtle pitfall: they don't track ownership. Anyone can `sem_post` even if they didn't `sem_wait`. With a mutex, only the holder should unlock. Some codebases ban semaphores for this reason; others use them for their conciseness.

## 17.7 Monitor pattern

A "monitor" (Hoare/Hansen) is a higher-level abstraction: a class whose methods are mutually exclusive. Java's `synchronized` and C# `lock` are monitor-style. In C with pthreads, you build it manually:

```c
typedef struct {
    pthread_mutex_t mu;
    pthread_cond_t cv;
    int state;
} monitor_t;

void monitor_op(monitor_t *m) {
    pthread_mutex_lock(&m->mu);
    while (!can_proceed(m->state))
        pthread_cond_wait(&m->cv, &m->mu);
    do_work(m);
    if (need_to_signal(m->state))
        pthread_cond_signal(&m->cv);
    pthread_mutex_unlock(&m->mu);
}
```

The discipline is to always pair `mu` and `cv`, always lock `mu` before checking or modifying `state`. Condition variables are the dirty implementation underneath the conceptual monitor.

## 17.8 Common bugs

1. **Missing wakeup**: signaler signals before waiter sleeps. Mitigated by checking the predicate while holding the mutex (waiter sees the change before sleeping).

2. **Lost wakeup**: consumer pre-empted between checking `count==0` and `cond_wait`. With `while + cond_wait` properly used, this doesn't happen — but using `if` instead of `while`, or signaling without holding the mutex, can cause it.

3. **Forgetting to broadcast on shutdown**: a "shutdown" signal needs to wake ALL waiters, not one. Use broadcast.

4. **Predicate change without signaling**: modifying state but forgetting to signal. Waiters never wake.

5. **Holding the lock too long**: blocks producers and consumers. Do work outside the critical section where possible.

## 17.9 Higher-level synchronization primitives

- **Barriers** (`pthread_barrier_t`): N threads must arrive before any proceeds. Useful in HPC and step-by-step parallel algorithms.
- **Latches / once** (`pthread_once`): an action runs exactly once, even if multiple threads call.
- **Events / Promises / Futures** (in C++ `std::future`, in Go channels): higher-level patterns built on condvars.

## Hands-on (30 minutes)

1. Implement and test the producer-consumer queue from §17.4. Run with multiple producers and consumers; verify items are produced and consumed correctly.

2. Replace `cond_signal` with `cond_broadcast` and run again. What changes? Should still be correct; possibly more wakeups.

3. Try the bug version: replace `while` with `if` in the wait loop. With high contention, run many iterations; eventually you may see anomalies (rare but possible). With ThreadSanitizer running, you might see warnings.

4. Implement the same producer-consumer with semaphores. Compare line counts and clarity.

5. Use a barrier to synchronize 4 threads:
   ```c
   pthread_barrier_t b;
   pthread_barrier_init(&b, NULL, 4);
   // each thread:
   do_phase1();
   pthread_barrier_wait(&b);
   do_phase2();
   pthread_barrier_wait(&b);
   ```
   Verify all threads finish phase 1 before any starts phase 2.

## Interview questions

### Q1. What is a condition variable and how does it differ from a mutex?

**Answer:** A condition variable is a synchronization primitive that lets a thread sleep until another thread signals that some condition might be true. It's always paired with a mutex.

The key difference from a mutex: a mutex provides mutual exclusion (only one thread inside a critical section). A condition variable provides waiting (sleep until signaled). They serve different purposes and are used together.

The canonical pattern:
```c
mutex_lock(&mu);
while (!condition)
    cond_wait(&cv, &mu);   // atomically unlock mu + sleep; on wake re-lock mu
do_work();
mutex_unlock(&mu);
```

The mutex protects the condition state; the cond_wait atomically releases the mutex and sleeps, so signalers that change the state and wake the cond_var don't lose the wakeup.

The check must be in a `while` loop, not `if`, to handle:
- **Spurious wakeups** (POSIX allows cond_wait to return without a signal).
- **Multiple waiters with broadcast**: all wake; the first to acquire the mutex may consume the resource; others find the condition false again.

Without proper use of cond_var, you'd either have to busy-wait (wasteful) or risk lost wakeups (bug).

### Q2. Why must you always check the predicate in a while loop in a cond_wait?

**Answer:** Two reasons:

1. **Spurious wakeups**: POSIX explicitly allows `pthread_cond_wait` to return without any signal having been sent. Implementations may do this for various reasons (signal interruptions, internal optimizations). If you proceed assuming the condition is true after a wake, your code is wrong.

2. **Multiple-waiter race**: when you use `pthread_cond_broadcast`, all waiting threads wake up. They re-acquire the mutex one at a time. The first thread might consume the resource (e.g., take the only item from a queue). When the second thread acquires the mutex, the queue is empty again — but it's running because of the broadcast. If it didn't re-check, it would proceed with the condition false.

Even with `signal` (one waiter wakes), there can be more subtle races: suppose two consumers and one producer. Producer signals; consumer A wakes; before A acquires the mutex, consumer B (already holding it) consumes the item; A acquires, finds empty, must wait again.

The discipline is universal: every `cond_wait` lives in `while (!condition) cond_wait(&cv, &mu);`. No exceptions. Many language libraries make this safer (e.g., C++'s `cv.wait(lk, predicate)` does the loop for you), but the underlying contract is the same.

### Q3. What's a semaphore? How does it relate to a mutex?

**Answer:** A semaphore is a non-negative integer with two atomic operations: `wait` (decrement; block if zero) and `post` (increment; wake one waiter if any). The integer represents available resources or pending events.

Semaphore vs. mutex:

- A **binary semaphore** (initialized to 1) acts like a mutex: wait acquires (decrement to 0), post releases (back to 1).
- A **counting semaphore** (initialized to N) allows up to N concurrent holders — useful for limiting concurrency on resource pools.
- A semaphore initialized to 0 acts like a "wait for event" — N posts wake N waiters in sequence.

Differences from a mutex:
- **No ownership tracking**: any thread can post; with a mutex, only the holder should unlock.
- **Can express counts**: a mutex is binary; a counting semaphore tracks how many resources are available.
- **Different mental model**: mutexes guard critical sections; semaphores represent resources.

When to choose semaphore vs. mutex+condition variable:
- **Resource pools** (max N downloads, fixed-size thread pool): semaphore is concise.
- **Producer-consumer with bounded buffer**: classic semaphore use case.
- **Arbitrary predicate** ("wait until at least 5 workers are registered AND no errors are pending"): mutex + condition variable is more flexible.

In modern C++ (`std::counting_semaphore`, C++20) and recent Linux APIs, semaphores are getting renewed attention. POSIX `sem_t` has been around forever (`sem_init`, `sem_wait`, `sem_post`).

### Q4. Implement a thread-safe blocking queue.

**Answer:** Sketch (production code would handle initialization, error checking, and likely use higher-level constructs):

```c
#define N 1024

typedef struct {
    int buffer[N];
    int head, tail, count;
    pthread_mutex_t mu;
    pthread_cond_t not_empty, not_full;
} bqueue_t;

void bqueue_init(bqueue_t *q) {
    q->head = q->tail = q->count = 0;
    pthread_mutex_init(&q->mu, NULL);
    pthread_cond_init(&q->not_empty, NULL);
    pthread_cond_init(&q->not_full, NULL);
}

void bqueue_push(bqueue_t *q, int item) {
    pthread_mutex_lock(&q->mu);
    while (q->count == N)
        pthread_cond_wait(&q->not_full, &q->mu);
    q->buffer[q->tail] = item;
    q->tail = (q->tail + 1) % N;
    q->count++;
    pthread_cond_signal(&q->not_empty);
    pthread_mutex_unlock(&q->mu);
}

int bqueue_pop(bqueue_t *q) {
    pthread_mutex_lock(&q->mu);
    while (q->count == 0)
        pthread_cond_wait(&q->not_empty, &q->mu);
    int item = q->buffer[q->head];
    q->head = (q->head + 1) % N;
    q->count--;
    pthread_cond_signal(&q->not_full);
    pthread_mutex_unlock(&q->mu);
    return item;
}
```

Key design decisions:
- **One mutex** protects all queue state.
- **Two condition variables**: producers wait on `not_full`, consumers on `not_empty`.
- **Always while-loop the wait** for spurious wakeups and broadcast races.
- **Signal after every modification**, so the other side sees the change.
- **Signal one** (not broadcast) since at most one thread can usefully proceed per modification.

For very high throughput, you could use lock-free queues (more complex; uses CAS), or split into separate lock-protected head/tail to allow concurrent producer and consumer (one at each end). The above is the textbook design — correct and clear, sufficient for most cases.

## Self-test

1. Why must `cond_wait` atomically release the mutex and go to sleep? What goes wrong if these are separate steps?
2. When would you use `cond_broadcast` instead of `cond_signal`?
3. A semaphore initialized to 1 is "like a mutex" except for what difference?
4. In a producer-consumer queue, can you eliminate the mutex if you use semaphores? Why or why not?
5. A thread does `pthread_cond_wait`. Another calls `pthread_cond_signal` before the first thread acquired the mutex protecting the predicate. What happens?
