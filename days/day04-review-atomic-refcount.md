# Day 4: Review & Refine — Thread-Safe Ref Counting with `std::atomic`

[← Back to Study Plan](../lld-study-plan.md) | [← Day 3](day03-shared-ptr.md)

> **Time**: ~2 hours
> **Goal**: Review D2-D3 implementations, fix edge cases, then upgrade `SharedPtr`'s control block to use `std::atomic` for thread-safe reference counting. Understand memory ordering.

---
---

# PART 1: THEORY — REVIEW CHECKLIST (15 min)

---

<br>

<h2 style="color: #2980B9;">📘 4.1 Review Your <code>UniquePtr</code> (D2)</h2>

Go through this checklist against your code:

| Check | Question | Fix if wrong |
|-------|----------|-------------|
| Copy deleted? | Are both copy ctor and copy assign `= delete` with `const`? | `UniquePtr(const UniquePtr&) = delete;` |
| Move noexcept? | Are move ctor and move assign marked `noexcept`? | STL containers fall back to copy without it |
| Move nulls source? | After move, is `other.ptr_` set to `nullptr`? | Prevents double delete |
| Constructor explicit? | Is `UniquePtr(T*)` marked `explicit`? | Prevents accidental ownership transfer |
| Self-move safe? | Does `p = std::move(p)` work? | `if (this != &other)` guard |
| `reset()` order? | Does `reset()` update `ptr_` before deleting old? | Prevents issues with `p.reset(p.get())` |
| `operator bool` explicit? | Is `operator bool()` marked `explicit`? | Prevents `int x = myPtr;` from compiling |
| Deleter stored efficiently? | Is the default deleter zero-cost (EBO)? | Inherit from `Deleter` instead of storing as member |

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 4.2 Review Your <code>SharedPtr</code> / <code>WeakPtr</code> (D3)</h2>

| Check | Question | Fix if wrong |
|-------|----------|-------------|
| Copy increments strong? | Does copy ctor do `++ctrl_->strong`? | Missing = premature delete |
| Destructor decrements? | Does destructor do `--ctrl_->strong` and delete on 0? | Missing = leak |
| Move doesn't touch count? | Move ctor steals pointers, no increment/decrement? | Incrementing on move = wrong count |
| Move nulls source? | Both `ptr_` and `ctrl_` set to `nullptr` on source? | Dangling control block access |
| Self-assignment safe? | Does `p = p` work without crash? | `if (this == &other) return *this;` |
| `WeakPtr::lock()` correct? | Returns empty `SharedPtr` if `expired()`? Increments strong if not? | Missing increment = use-after-free |
| Control block lifetime? | Block deleted only when strong==0 AND weak==0? | Premature delete = WeakPtr crashes |
| No double control block? | Two `SharedPtr`s from same raw pointer → crash. Is this documented/tested? | Always copy/move from existing `SharedPtr` |

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 4.3 Edge Cases to Test</h2>

Run all of these under `-fsanitize=address`:

```cpp
// 1. Empty SharedPtr
SharedPtr<int> empty;
assert(empty.get() == nullptr);
assert(empty.use_count() == 0);

// 2. Self-assignment
auto p = SharedPtr<int>(new int(42));
p = p;
assert(*p == 42);
assert(p.use_count() == 1);

// 3. Chain of copies then destruction
{
    auto a = SharedPtr<int>(new int(10));
    auto b = a;
    auto c = b;
    assert(a.use_count() == 3);
}  // all destroyed — no leaks

// 4. WeakPtr outlives SharedPtr
WeakPtr<int> w;
{
    auto sp = SharedPtr<int>(new int(99));
    w = sp;
    assert(!w.expired());
}
assert(w.expired());
auto locked = w.lock();
assert(locked.get() == nullptr);

// 5. Move from SharedPtr
auto x = SharedPtr<int>(new int(7));
auto y = std::move(x);
assert(x.get() == nullptr);
assert(x.use_count() == 0);
assert(*y == 7);

// 6. reset() on last owner
auto r = SharedPtr<int>(new int(55));
r.reset();
assert(r.get() == nullptr);
```

<br><br>

---
---

# PART 2: THEORY — ATOMIC REFERENCE COUNTING (25 min)

---

<br>

<h2 style="color: #2980B9;">📘 4.4 Why Do We Need Atomics?</h2>

Your current `ControlBlock` uses plain `size_t` for reference counts:

```cpp
size_t strong;
size_t weak;
```

This is a **data race** if two threads copy/destroy `SharedPtr`s to the same object concurrently:

```
Thread A: copies sp → reads strong (1) → writes strong (2)
Thread B: destroys sp → reads strong (1) → writes strong (0) → deletes object

Both read 1 simultaneously. Thread A writes 2, Thread B writes 0.
Final value depends on who writes last — could be 0 (object deleted while A still uses it)
or 2 (object never deleted — leak).
```

This is undefined behavior. The fix: make increment/decrement **atomic**.

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 4.4a Deep Dive: What Exactly Is a Data Race? (Visual)</h2>

To understand atomics, you first need to see **why normal variables break** in multithreading.

<br>

#### How `++counter` actually works on the CPU

When you write `counter++` in C++, it looks like one operation. But the CPU actually does **three steps**:

```
   Your C++ code:     counter++;

   What the CPU does:
     Step 1: LOAD   — read counter from memory into a CPU register
     Step 2: ADD    — add 1 to the register
     Step 3: STORE  — write the register back to memory
```

This is called a **read-modify-write** operation. The problem: another thread can interfere between any of these steps.

<br>

#### The data race — step by step

Suppose `counter = 5` and two threads both do `counter++`:

```
                    Memory: counter = 5

Thread A                              Thread B
────────                              ────────
LOAD counter → reg_A = 5
                                      LOAD counter → reg_B = 5
ADD 1        → reg_A = 6
                                      ADD 1        → reg_B = 6
STORE reg_A  → counter = 6
                                      STORE reg_B  → counter = 6

                    Memory: counter = 6   ← WRONG! Should be 7
```

Both threads read 5, both computed 6, both wrote 6. One increment was **lost**. This is called a **lost update** or **torn read/write**.

<br>

#### The same race with ref counting

Now apply this to `SharedPtr`. `strong` starts at 1:

```
                    Memory: strong = 1

Thread A (copying sp)                 Thread B (destroying sp)
─────────────────────                 ──────────────────────────
LOAD strong → reg_A = 1
                                      LOAD strong → reg_B = 1
ADD 1       → reg_A = 2
                                      SUB 1       → reg_B = 0
STORE reg_A → strong = 2
                                      STORE reg_B → strong = 0
                                      strong == 0 → DELETE object! 💥

Thread A now holds a SharedPtr to DELETED memory → use-after-free
```

Or the reverse order:

```
                    Memory: strong = 1

Thread A (copying sp)                 Thread B (destroying sp)
─────────────────────                 ──────────────────────────
LOAD strong → reg_A = 1
                                      LOAD strong → reg_B = 1
                                      SUB 1       → reg_B = 0
                                      STORE reg_B → strong = 0
                                      strong == 0 → DELETE object! 💥
ADD 1       → reg_A = 2
STORE reg_A → strong = 2             ← writes 2 to memory, but object is already deleted!

Thread A: strong says 2, but the object is GONE → undefined behavior
```

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 4.4b How Atomics Fix This</h2>

`std::atomic<size_t>` guarantees that the **entire read-modify-write** happens as ONE indivisible step. No other thread can see the intermediate state.

```
                    Memory: counter = 5

Thread A                              Thread B
────────                              ────────
ATOMIC_ADD(counter, 1) ─────┐
   ┌─ LOAD  = 5             │
   ├─ ADD 1 = 6             │    (Thread B is BLOCKED — hardware ensures
   └─ STORE = 6 ────────────┘     it cannot access counter during this)
                                      
                                  ATOMIC_ADD(counter, 1) ─────┐
                                     ┌─ LOAD  = 6             │
                                     ├─ ADD 1 = 7             │
                                     └─ STORE = 7 ────────────┘

                    Memory: counter = 7   ← CORRECT!
```

The "BLOCKED" isn't literally a lock (that would be a mutex). At the hardware level, the CPU uses special instructions:
- **x86**: `lock add` — the `lock` prefix makes the entire instruction atomic
- **ARM**: `ldxr` / `stxr` (load-exclusive / store-exclusive) — a loop that retries if another core touched the memory

But from your perspective, you just write:

```cpp
std::atomic<size_t> counter{5};
counter.fetch_add(1);  // one atomic operation — guaranteed indivisible
```

<br>

#### Visual: Atomic vs Non-atomic

```
Non-atomic counter++:
  ┌──────┐    ┌──────┐    ┌──────┐
  │ LOAD │ →  │ ADD  │ →  │STORE │     3 separate steps — another thread
  └──────┘    └──────┘    └──────┘     can interfere between any two

Atomic fetch_add(1):
  ┌─────────────────────────────────┐
  │    LOAD  →  ADD  →  STORE      │   ALL happen as one unit — no thread
  └─────────────────────────────────┘   can see the intermediate state
```

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 4.4c Memory Layout: Where Do Atomics Live?</h2>

An `std::atomic<size_t>` is just a `size_t` in memory — it's the same 8 bytes at the same address. The "atomic" part is not about the storage, it's about the **instructions** used to access it.

```
Regular size_t:               std::atomic<size_t>:
┌────────────────┐            ┌────────────────┐
│   8 bytes      │            │   8 bytes      │     ← same size, same layout
│   value: 42    │            │   value: 42    │
└────────────────┘            └────────────────┘
  accessed with:                accessed with:
  MOV (regular load/store)      LOCK ADD, XCHG, CMPXCHG
                                (special atomic CPU instructions)
```

So `sizeof(std::atomic<size_t>) == sizeof(size_t)`. No extra overhead in storage.

The ControlBlock in memory looks like:

```
ControlBlock<Widget> (on the heap)
┌──────────────────────────────────────────────────┐
│  T* ptr            [8 bytes]  → points to Widget │
│  atomic<size_t> strong [8 bytes]  value: 2       │
│  atomic<size_t> weak   [8 bytes]  value: 1       │
└──────────────────────────────────────────────────┘
              ↑                           │
              │                           ▼
   SharedPtr A (stack)              ┌───────────┐
   ┌─────────────┐                 │  Widget    │
   │ ptr_  ──────────────────────► │  id: 42    │
   │ ctrl_ ──────┘                 └───────────┘
   └─────────────┘

   SharedPtr B (stack, another thread)
   ┌─────────────┐
   │ ptr_  ──────────────────────► (same Widget)
   │ ctrl_ ──────┘ (same ControlBlock)
   └─────────────┘
```

When Thread A does `auto b = a;` (copy):
- `b.ptr_` = `a.ptr_` (just pointer copy, no atomic needed)
- `b.ctrl_` = `a.ctrl_` (just pointer copy, no atomic needed)
- `ctrl_->strong.fetch_add(1)` ← THIS is the atomic part (strong: 1 → 2)

When Thread B destroys its `SharedPtr`:
- `ctrl_->strong.fetch_sub(1)` ← atomic (strong: 2 → 1, returns old value 2, not 1, so don't delete)

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 4.4d Memory Ordering — The Full Picture with Diagrams</h2>

Atomics solve the **indivisibility** problem. But there's a second problem: **reordering**.

<br>

#### The reordering problem

Modern CPUs and compilers **reorder instructions** for performance. Your code:

```cpp
data = 42;            // write to regular variable
ready.store(true);    // write to atomic
```

The CPU might actually execute:

```cpp
ready.store(true);    // atomic write happens FIRST
data = 42;            // regular write happens SECOND
```

This is catastrophic:

```
Thread A (producer)                   Thread B (consumer)
───────────────────                   ───────────────────
data = 42;
ready.store(true);     ──────────►    if (ready.load()) {
                                          use(data);  // might see GARBAGE
                                          // because data=42 hasn't happened yet!
                                      }
```

Memory ordering tells the CPU: "don't reorder past this point."

<br>

#### The 3 orderings you need to know (with diagrams)

**`memory_order_relaxed` — No ordering, just atomicity**

```
Thread A:
  ┌──────────────────────┐
  │ x = 1;               │  ← can move below the atomic
  │ y = 2;               │  ← can move below the atomic
  │ counter.fetch_add(1, │
  │   relaxed);          │  ← only guarantees THIS operation is atomic
  │ z = 3;               │  ← can move above the atomic
  └──────────────────────┘

  The CPU can reorder x, y, z freely around the atomic.
  Only the atomic itself (fetch_add) is guaranteed indivisible.
```

Use for: incrementing ref count (you just need the number to be correct, don't care about ordering of other writes).

<br>

**`memory_order_release` — "Publish my writes"**

```
Thread A (writer):
  ┌──────────────────────┐
  │ data.name = "Alice";  │  ← these writes are FLUSHED
  │ data.age = 30;        │  ← to memory BEFORE the store
  │ ════════════════════  │  ← RELEASE BARRIER (nothing above can move below)
  │ flag.store(true,      │
  │   release);           │
  └──────────────────────┘
```

Everything written BEFORE the release-store is guaranteed to be visible to any thread that does an acquire-load on the same atomic.

<br>

**`memory_order_acquire` — "See the publisher's writes"**

```
Thread B (reader):
  ┌──────────────────────┐
  │ if (flag.load(        │
  │   acquire)) {         │
  │ ════════════════════  │  ← ACQUIRE BARRIER (nothing below can move above)
  │   use(data.name);     │  ← guaranteed to see "Alice"
  │   use(data.age);      │  ← guaranteed to see 30
  │ }                     │
  └──────────────────────┘
```

<br>

**Together — the acquire-release pair:**

```
Thread A                              Thread B
────────                              ────────
  obj->value = 100;                     
  obj->name = "test";                   
  ═══ RELEASE ═══                       
  strong.fetch_sub(1, release);         
                                        strong.fetch_sub(1, acquire);
                                        ═══ ACQUIRE ═══
                                        // if strong hit 0, Thread B deletes
                                        // it is GUARANTEED to see value=100
                                        // and name="test" from Thread A
                                        delete obj;  // safe!
```

Without acquire-release, Thread B might delete the object while Thread A's writes (`value = 100`) are still sitting in Thread A's CPU cache, not yet visible to Thread B. The object would be deleted before Thread A's modifications are actually committed to shared memory.

<br>

#### Applied to ref counting

```
Increment (copy SharedPtr) — relaxed is fine:
  ┌─────────────────────────────────────────┐
  │ "I'm joining as an owner.               │
  │  The object is already alive.            │
  │  I don't need to see anyone else's       │
  │  writes right now — I'll access the      │
  │  object later through my own pointer."   │
  │                                          │
  │  strong.fetch_add(1, relaxed);           │
  └─────────────────────────────────────────┘

Decrement (destroy SharedPtr) — acq_rel needed:
  ┌─────────────────────────────────────────┐
  │ "I'm leaving. I might be the last one.  │
  │                                          │
  │  RELEASE: flush all my writes to the     │
  │  object so the deleter can see them.     │
  │                                          │
  │  ACQUIRE: if I'm the last one (count→0),│
  │  I need to see all writes from every     │
  │  other thread that released before me."  │
  │                                          │
  │  strong.fetch_sub(1, acq_rel);           │
  │  if (old_value == 1) → delete obj;       │
  └─────────────────────────────────────────┘
```

<br>

#### The "party" analogy

```
RELAXED increment = "I'm arriving at the party"
  → Just need the headcount to go up.
  → Don't need to see what dishes others are cooking.

ACQ_REL decrement = "I'm leaving the party, might need to clean up"
  → RELEASE: put my dishes in the sink (make my writes visible).
  → ACQUIRE: if I'm the last to leave, I need to see ALL the dishes
    from everyone who already left, so I can clean everything up.

SEQ_CST = "a strict bouncer who enforces a single queue"
  → Everyone arrives and leaves in one agreed-upon global order.
  → Slowest but impossible to get wrong.
```

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 4.4e Hands-On: See the Data Race Yourself</h2>

Create this test to observe a real data race:

```cpp
#include <iostream>
#include <thread>
#include <vector>

// NON-ATOMIC — will produce wrong results
size_t counter = 0;

void increment_many() {
    for (int i = 0; i < 1000000; ++i) {
        counter++;   // not atomic — data race!
    }
}

int main() {
    std::vector<std::thread> threads;
    for (int i = 0; i < 4; ++i)
        threads.emplace_back(increment_many);
    for (auto& t : threads) t.join();

    std::cout << "Expected: " << 4 * 1000000 << "\n";
    std::cout << "Actual:   " << counter << "\n";
    // You'll see something like 2,347,891 instead of 4,000,000
    return 0;
}
```

Now fix it with `std::atomic`:

```cpp
#include <atomic>

std::atomic<size_t> counter{0};  // only change: atomic

void increment_many() {
    for (int i = 0; i < 1000000; ++i) {
        counter.fetch_add(1, std::memory_order_relaxed);  // atomic increment
    }
}
// Now always prints 4,000,000
```

Compile and compare:

```bash
# Non-atomic (will show wrong count, TSan will flag it)
g++ -std=c++17 -O2 -fsanitize=thread -o race race.cpp && ./race

# Atomic (correct count, TSan is happy)
g++ -std=c++17 -O2 -fsanitize=thread -o no_race no_race.cpp && ./no_race
```

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 4.5 What Is <code>std::atomic</code>?</h2>

`std::atomic<T>` wraps a value and guarantees that reads and writes are **indivisible** — no other thread can see a half-written value.

```cpp
#include <atomic>

std::atomic<size_t> counter{0};

counter++;          // atomic increment — safe from any thread
counter--;          // atomic decrement
size_t val = counter.load();   // atomic read
counter.store(5);              // atomic write
```

Key operations for ref counting:

| Operation | What it does | Return value |
|-----------|-------------|--------------|
| `++counter` | Atomically increment, return new value | new value |
| `--counter` | Atomically decrement, return new value | new value |
| `counter.fetch_add(1)` | Atomically increment, return **old** value | old value |
| `counter.fetch_sub(1)` | Atomically decrement, return **old** value | old value |
| `counter.load()` | Atomically read | current value |
| `counter.store(n)` | Atomically write | void |

For ref counting, `fetch_sub(1)` is preferred because you need the **old** value to check if it was 1 (meaning you just decremented to 0):

```cpp
if (counter.fetch_sub(1) == 1) {
    // counter was 1, now it's 0 → last owner → delete
}
```

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 4.6 Memory Ordering — Why It Matters</h2>

Every atomic operation has an optional **memory order** parameter that controls how much the CPU/compiler can reorder surrounding reads and writes.

```cpp
counter.fetch_sub(1, std::memory_order_acq_rel);
//                    ^^^^^^^^^^^^^^^^^^^^^^^^
//                    this part controls ordering
```

There are 6 memory orders, but for ref counting you only need to understand 3:

<br>

#### `memory_order_relaxed` — No ordering guarantees

The atomic operation itself is indivisible, but surrounding non-atomic reads/writes can be reordered freely. Cheapest.

```cpp
counter.fetch_add(1, std::memory_order_relaxed);  // just need the count to be correct
```

Good for **incrementing** the ref count — when you're copying a `SharedPtr`, you just need the count to go up. You don't need other threads to see your non-atomic writes in any particular order at this point.

<br>

#### `memory_order_acquire` — "I want to see everything the releasing thread did"

When you **load/read** with acquire, you're guaranteed to see all writes that happened **before** the corresponding release in another thread.

<br>

#### `memory_order_release` — "Make my writes visible to the acquiring thread"

When you **store/write** with release, all your preceding writes become visible to any thread that does an acquire load on the same atomic.

<br>

#### `memory_order_acq_rel` — Both acquire AND release

Used for **read-modify-write** operations (like `fetch_sub`) where you both read the old value and write the new value atomically.

<br>

#### What `std::shared_ptr` actually uses

```
Incrementing strong count (copy):     memory_order_relaxed
   → just bump the number, no ordering needed

Decrementing strong count (destroy):  memory_order_acq_rel
   → need to ensure all writes to the managed object by this thread are visible
     to whichever thread ends up deleting the object (the last one to decrement)

After decrement hits 0 (before delete): memory_order_acquire fence
   → ensure we see all writes from all other threads that previously decremented
```

**For your learning implementation, using the default `memory_order_seq_cst` (sequential consistency) is perfectly fine.** It's the strongest and slowest ordering, but it's correct and easiest to reason about. The relaxed/acq_rel optimization is a production concern.

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 4.7 <code>fetch_sub</code> Returns the OLD Value — Why This Matters</h2>

This is a common source of bugs:

```cpp
// WRONG — race condition:
--counter;
if (counter == 0) {   // another thread might decrement between these two lines!
    delete obj;
}

// CORRECT — single atomic operation:
if (counter.fetch_sub(1) == 1) {   // fetch_sub returns OLD value
    // old value was 1, meaning we just made it 0 → we're the last owner
    delete obj;
}
```

With `fetch_sub`, the decrement and the read of the old value are **one atomic operation** — no window for another thread to interfere.

Why check `== 1` and not `== 0`? Because `fetch_sub` returns the value **before** subtraction. If it was 1, after subtraction it's 0.

<br><br>

---
---

# PART 3: BUILD EXERCISE — UPGRADE TO ATOMIC (45 min)

---

<br>

## Exercise: Make Your ControlBlock Thread-Safe

<br>

### Step 1: Change the count types

```cpp
#include <atomic>

template <typename T>
class ControlBlock {
public:
    T* ptr;
    std::atomic<size_t> strong;
    std::atomic<size_t> weak;

    explicit ControlBlock(T* p) : ptr(p), strong(1), weak(0) {}
};
```

`std::atomic<size_t>` cannot be copied or moved, so make sure your `ControlBlock` is never copied (it shouldn't be — it's always heap-allocated and shared via pointer).

<br>

### Step 2: Update increment operations

Everywhere you had `++ctrl_->strong`, use atomic increment:

```cpp
// Copy constructor — increment strong count
SharedPtr(const SharedPtr& other)
    : ptr_(other.ptr_), ctrl_(other.ctrl_)
{
    if (ctrl_) ctrl_->strong.fetch_add(1, std::memory_order_relaxed);
}
```

Why `relaxed`? When copying, we just need the count to go up. We don't need ordering guarantees — the object is already alive (someone else has a strong ref), and we're just adding ourselves.

<br>

### Step 3: Update decrement operations (the critical part)

```cpp
void release() {
    if (!ctrl_) return;

    // Decrement strong. If we were the last strong owner, destroy the object.
    if (ctrl_->strong.fetch_sub(1, std::memory_order_acq_rel) == 1) {
        delete ctrl_->ptr;
        ctrl_->ptr = nullptr;

        // If also no weak refs, delete the control block
        if (ctrl_->weak.load(std::memory_order_acquire) == 0) {
            delete ctrl_;
        }
    }

    ptr_ = nullptr;
    ctrl_ = nullptr;
}
```

Why `acq_rel` on the decrement?
- **Release**: our writes to the managed object (through `*ptr_`) must be visible to whoever ends up deleting it
- **Acquire**: if we're the one deleting, we must see all writes from all other threads that released before us

<br>

### Step 4: Update WeakPtr similarly

```cpp
void release_weak() {
    if (!ctrl_) return;
    if (ctrl_->weak.fetch_sub(1, std::memory_order_acq_rel) == 1) {
        // Last weak ref gone. If strong is also 0, delete block.
        if (ctrl_->strong.load(std::memory_order_acquire) == 0) {
            delete ctrl_;
        }
    }
    ctrl_ = nullptr;
}
```

<br>

### Step 5: Fix `WeakPtr::lock()` — the tricky one

`lock()` must atomically check if strong > 0 and increment it. A simple `if (strong > 0) ++strong` has a race — another thread could decrement to 0 between your check and increment.

Use a **compare-and-swap loop**:

```cpp
SharedPtr<T> lock() const {
    if (!ctrl_) return SharedPtr<T>();

    size_t count = ctrl_->strong.load(std::memory_order_relaxed);
    while (count > 0) {
        // Try to atomically change strong from 'count' to 'count+1'
        if (ctrl_->strong.compare_exchange_weak(
                count, count + 1, std::memory_order_acq_rel)) {
            // Success — we incremented strong. Build a SharedPtr.
            return SharedPtr<T>(ctrl_, ctrl_->ptr);
        }
        // Failed — 'count' was updated to the current value by compare_exchange_weak.
        // Loop will re-check if it's still > 0.
    }
    return SharedPtr<T>();  // strong hit 0 — object is gone
}
```

This is your first **CAS (Compare-And-Swap) loop** — a fundamental lock-free pattern you'll use heavily in Week 5.

<br>

#### How `compare_exchange_weak` works

```cpp
bool compare_exchange_weak(T& expected, T desired);
```

Atomically:
1. Read the current value of the atomic
2. If it equals `expected` → write `desired`, return `true`
3. If it doesn't → update `expected` to the actual current value, return `false`

The "weak" variant may **spuriously fail** (return false even if the value matches) — that's fine because we're in a loop anyway. It's slightly faster than `compare_exchange_strong` on some architectures (ARM).

<br>

### Full test for thread safety:

```cpp
#include <thread>
#include <vector>

void stress_test() {
    auto sp = SharedPtr<int>(new int(42));

    auto worker = [sp]() mutable {   // each thread gets a copy (increments strong)
        for (int i = 0; i < 100000; ++i) {
            auto copy = sp;          // increment
            (void)copy;              // decrement when copy goes out of scope
        }
    };

    std::vector<std::thread> threads;
    for (int i = 0; i < 8; ++i) {
        threads.emplace_back(worker);
    }
    for (auto& t : threads) t.join();

    std::cout << "Final use_count: " << sp.use_count() << "\n";  // should be 1
    std::cout << "Value: " << *sp << "\n";                        // should be 42
}
```

Compile with ThreadSanitizer:

```bash
g++ -std=c++17 -Wall -Wextra -fsanitize=thread -o day04 day04_atomic.cpp && ./day04
```

If you still have data races, TSan will tell you exactly which lines are involved.

<br><br>

---
---

# PART 4: DEEP DIVE Q&A

---

<br>

<h2 style="color: #2980B9;">📘 Q1: Why is <code>relaxed</code> okay for increment but not decrement?</h2>

**Incrementing** (copying a `SharedPtr`):
- The object is already alive — someone else holds a strong ref
- You're just adding yourself as an owner
- You don't need to see anyone else's writes to the object yet
- So `relaxed` is sufficient — just make the count correct

**Decrementing** (destroying a `SharedPtr`):
- You might be the **last** owner
- If you are, you'll `delete` the object
- Before deleting, you must see **all modifications** other threads made to the object through their `SharedPtr`s
- So you need `acquire` (to see their writes) and `release` (to make your writes visible to whoever deletes)
- Hence `acq_rel`

Think of it this way: incrementing is "joining the party" (no coordination needed), decrementing is "possibly turning off the lights" (you need to make sure everyone has left).

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 Q2: Why <code>compare_exchange_weak</code> in <code>lock()</code> instead of just <code>fetch_add</code>?</h2>

If you used `fetch_add`:

```cpp
// WRONG:
SharedPtr<T> lock() const {
    if (ctrl_->strong.load() > 0) {     // check
        ctrl_->strong.fetch_add(1);      // increment
        return SharedPtr<T>(ctrl_, ctrl_->ptr);
    }
    return SharedPtr<T>();
}
```

**Race condition**: Between the `load()` and the `fetch_add()`, another thread could decrement strong to 0 and delete the object. Your `fetch_add` would resurrect a dead count from 0 to 1, and you'd return a `SharedPtr` to deleted memory.

The CAS loop avoids this because the check and the increment are **one atomic operation**:

```cpp
// "Only increment if the current value is what I expect (and > 0)"
compare_exchange_weak(count, count + 1)
```

If another thread changed the count between your load and your CAS attempt, CAS fails and you re-read the actual value. If it's now 0, you exit the loop and return empty.

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 Q3: What is the difference between <code>compare_exchange_weak</code> and <code>compare_exchange_strong</code>?</h2>

Both do the same thing conceptually:
- If `atomic == expected` → write `desired`, return `true`
- If `atomic != expected` → update `expected` to actual value, return `false`

The difference:

| | `weak` | `strong` |
|---|--------|----------|
| May spuriously fail? | **Yes** — can return `false` even when `atomic == expected` | **No** — only fails when values actually differ |
| Performance | Slightly faster on ARM/LL-SC architectures | Slightly slower (internally loops to eliminate spurious failures) |
| When to use | Always inside a loop (spurious failure is just another iteration) | When you need a single try (rare — most CAS is in loops) |

Since `lock()` already has a `while` loop, `weak` is the right choice — a spurious failure just means one extra loop iteration, and it's faster per attempt.

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 Q4: Can <code>std::atomic</code> be copied or moved?</h2>

**No.** `std::atomic<T>` has its copy constructor and copy assignment **deleted**. This is intentional — what would it mean to "copy" an atomic? The copy operation itself can't be atomic (you'd need to atomically read the source AND write the destination as one operation, which hardware doesn't support for two separate memory locations).

This means:
- `ControlBlock` can't be copied or moved (which is correct — it lives on the heap and is shared by pointer)
- You can't put `std::atomic` in a `std::vector` directly
- You can't return `std::atomic` by value from a function

If you need to read the value, use `.load()`. If you need to write, use `.store()`.

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 Q5: Does <code>memory_order_seq_cst</code> (the default) work everywhere?</h2>

Yes. `seq_cst` (sequentially consistent) is the **default** for all atomic operations and is the strongest ordering. It guarantees a single total order of all atomic operations across all threads — as if they happened one at a time in some global sequence.

```cpp
counter.fetch_sub(1);   // same as: counter.fetch_sub(1, std::memory_order_seq_cst);
```

**For learning, always use `seq_cst` (the default)**. It's correct and easiest to reason about.

The weaker orderings (`relaxed`, `acq_rel`) are **optimizations** — they let the CPU reorder more instructions for performance. On x86, the difference is small (x86 is already strongly ordered). On ARM, the difference can be significant.

**Rule for interviews**: Know that weaker orderings exist and why they matter. Be able to explain why increment uses `relaxed` and decrement uses `acq_rel`. But don't try to use them unless you're 100% sure — `seq_cst` is always safe.

<br><br>

---
---

# PART 5: REFLECT & NOTES (10 min)

---

<br>

1. **What happens if you use plain `size_t` instead of `std::atomic<size_t>` for ref counts?**
   - Data race → undefined behavior. Counts can be corrupted, leading to double-free or leaks.

2. **Why does `fetch_sub` return the OLD value?**
   - So you can check if you were the last owner (`old == 1` means you just decremented to 0) in a single atomic operation with no race window.

3. **What is a CAS loop?**
   - Compare-And-Swap loop: read current value, compute desired value, attempt to atomically swap. If another thread changed the value, retry. This is the foundation of lock-free programming (Week 5).

4. **When would you use `memory_order_relaxed`?**
   - When you just need the atomic operation to be indivisible, but don't care about ordering of surrounding non-atomic operations. E.g., incrementing a ref count during copy.

5. **Can you make the managed object (`T`) thread-safe by using `shared_ptr`?**
   - No. `shared_ptr` only makes the **ref count** thread-safe. Access to `*ptr` still needs your own synchronization (mutex, atomic members, etc.).

<br><br>

---
---

# INTERVIEW QUESTIONS TO PRACTICE

---

<br>

1. "Is `shared_ptr` thread-safe? What exactly is thread-safe about it?"
2. "Explain the memory ordering used in `shared_ptr`'s reference counting."
3. "Implement a thread-safe `lock()` for `weak_ptr` using CAS."
4. "What is `compare_exchange_weak` and when would you use it?"
5. "What happens if two threads simultaneously destroy the last two `shared_ptr`s to the same object?"
6. "Why can't `std::atomic` be copied?"

<br><br>

---
---

# REFERENCE SOLUTION

---

<br>

<details>
<summary>Click to expand the atomic ControlBlock + thread-safe SharedPtr</summary>

<br>

```cpp
#include <iostream>
#include <atomic>
#include <utility>
#include <thread>
#include <vector>
#include <cassert>

template <typename T> class SharedPtr;
template <typename T> class WeakPtr;

template <typename T>
struct ControlBlock {
    T* ptr;
    std::atomic<size_t> strong;
    std::atomic<size_t> weak;

    explicit ControlBlock(T* p) : ptr(p), strong(1), weak(0) {}
};

template <typename T>
class SharedPtr {
    friend class WeakPtr<T>;

    T* ptr_ = nullptr;
    ControlBlock<T>* ctrl_ = nullptr;

    SharedPtr(ControlBlock<T>* c, T* p) : ptr_(p), ctrl_(c) {
        // Used by WeakPtr::lock — strong already incremented by CAS
    }

    void release() {
        if (!ctrl_) return;
        if (ctrl_->strong.fetch_sub(1, std::memory_order_acq_rel) == 1) {
            delete ctrl_->ptr;
            ctrl_->ptr = nullptr;
            if (ctrl_->weak.load(std::memory_order_acquire) == 0) {
                delete ctrl_;
            }
        }
        ptr_ = nullptr;
        ctrl_ = nullptr;
    }

public:
    SharedPtr() = default;
    explicit SharedPtr(T* raw) : ptr_(raw) {
        if (raw) ctrl_ = new ControlBlock<T>(raw);
    }

    ~SharedPtr() { release(); }

    SharedPtr(const SharedPtr& o) : ptr_(o.ptr_), ctrl_(o.ctrl_) {
        if (ctrl_) ctrl_->strong.fetch_add(1, std::memory_order_relaxed);
    }

    SharedPtr& operator=(const SharedPtr& o) {
        if (this == &o) return *this;
        release();
        ptr_ = o.ptr_;
        ctrl_ = o.ctrl_;
        if (ctrl_) ctrl_->strong.fetch_add(1, std::memory_order_relaxed);
        return *this;
    }

    SharedPtr(SharedPtr&& o) noexcept : ptr_(o.ptr_), ctrl_(o.ctrl_) {
        o.ptr_ = nullptr;
        o.ctrl_ = nullptr;
    }

    SharedPtr& operator=(SharedPtr&& o) noexcept {
        if (this == &o) return *this;
        release();
        ptr_ = o.ptr_;
        ctrl_ = o.ctrl_;
        o.ptr_ = nullptr;
        o.ctrl_ = nullptr;
        return *this;
    }

    T* get() const { return ptr_; }
    T& operator*() const { return *ptr_; }
    T* operator->() const { return ptr_; }
    explicit operator bool() const { return ptr_ != nullptr; }

    size_t use_count() const {
        return ctrl_ ? ctrl_->strong.load(std::memory_order_relaxed) : 0;
    }

    void reset(T* raw = nullptr) {
        release();
        if (raw) {
            ptr_ = raw;
            ctrl_ = new ControlBlock<T>(raw);
        }
    }
};

template <typename T>
class WeakPtr {
    ControlBlock<T>* ctrl_ = nullptr;

    void release_weak() {
        if (!ctrl_) return;
        if (ctrl_->weak.fetch_sub(1, std::memory_order_acq_rel) == 1) {
            if (ctrl_->strong.load(std::memory_order_acquire) == 0) {
                delete ctrl_;
            }
        }
        ctrl_ = nullptr;
    }

public:
    WeakPtr() = default;

    WeakPtr(const SharedPtr<T>& sp) : ctrl_(sp.ctrl_) {
        if (ctrl_) ctrl_->weak.fetch_add(1, std::memory_order_relaxed);
    }

    WeakPtr(const WeakPtr& o) : ctrl_(o.ctrl_) {
        if (ctrl_) ctrl_->weak.fetch_add(1, std::memory_order_relaxed);
    }

    WeakPtr& operator=(const WeakPtr& o) {
        if (this == &o) return *this;
        release_weak();
        ctrl_ = o.ctrl_;
        if (ctrl_) ctrl_->weak.fetch_add(1, std::memory_order_relaxed);
        return *this;
    }

    ~WeakPtr() { release_weak(); }

    bool expired() const {
        return !ctrl_ || ctrl_->strong.load(std::memory_order_acquire) == 0;
    }

    SharedPtr<T> lock() const {
        if (!ctrl_) return SharedPtr<T>();
        size_t count = ctrl_->strong.load(std::memory_order_relaxed);
        while (count > 0) {
            if (ctrl_->strong.compare_exchange_weak(
                    count, count + 1, std::memory_order_acq_rel)) {
                return SharedPtr<T>(ctrl_, ctrl_->ptr);
            }
        }
        return SharedPtr<T>();
    }
};

// --- Tests ---

struct Widget {
    int id;
    Widget(int i) : id(i) { std::cout << "  Widget(" << id << ") born\n"; }
    ~Widget() { std::cout << "  Widget(" << id << ") died\n"; }
};

int main() {
    std::cout << "=== Basic test ===\n";
    {
        auto a = SharedPtr<Widget>(new Widget(1));
        auto b = a;
        std::cout << "  use_count: " << a.use_count() << "\n";
    }

    std::cout << "\n=== WeakPtr test ===\n";
    {
        WeakPtr<Widget> w;
        {
            auto sp = SharedPtr<Widget>(new Widget(2));
            w = sp;
            std::cout << "  expired inside: " << w.expired() << "\n";
            auto locked = w.lock();
            std::cout << "  locked id: " << locked->id << "\n";
        }
        std::cout << "  expired outside: " << w.expired() << "\n";
    }

    std::cout << "\n=== Thread stress test ===\n";
    {
        auto sp = SharedPtr<int>(new int(42));
        auto worker = [sp]() mutable {
            for (int i = 0; i < 100000; ++i) {
                auto copy = sp;
                (void)copy;
            }
        };
        std::vector<std::thread> threads;
        for (int i = 0; i < 8; ++i)
            threads.emplace_back(worker);
        for (auto& t : threads) t.join();
        std::cout << "  Final use_count: " << sp.use_count() << "\n";
        std::cout << "  Value: " << *sp << "\n";
    }

    std::cout << "\n=== Done ===\n";
    return 0;
}
```

</details>

<br>

---

**Next**: [Day 5 — RAII: ScopeGuard & FileHandle →](day05-raii-scope-guard.md)  *(coming soon)*
