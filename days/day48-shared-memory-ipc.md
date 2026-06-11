# Day 48: Shared Memory IPC

[← Back to Study Plan](../lld-study-plan.md) | [← Day 47](day47-serialization.md)

> **Time**: ~2-3 hours (weekend day)
> **Goal**: Sockets (Days 45-47) move bytes between processes by copying through the kernel. **Shared memory** is the fastest IPC there is: two processes map the *same physical pages* into their address spaces and read/write them directly — zero copies, no syscall on the data path. Learn the POSIX shared-memory API (`shm_open` → `ftruncate` → `mmap`), **named vs anonymous** mappings, the hard part — **cross-process synchronization** with a `pthread_mutex` marked `PTHREAD_PROCESS_SHARED` or with **atomics**, **memory ordering across processes** (the same acquire/release rules as threads, because it's the same hardware), and **lifetime/cleanup** (`shm_unlink`, the leak that survives your process). Build a single-producer/single-consumer **ring buffer in shared memory** passing messages between two separate processes.

---
---

# PART 1: WHAT SHARED MEMORY IS

---
---

<br>

<h2 style="color: #2980B9;">📘 48.1 Zero-Copy IPC</h2>

Every other IPC mechanism copies data through the kernel: a pipe or socket `write` copies your bytes into a kernel buffer, and the reader's `read` copies them back out — two copies and two syscalls per message. Shared memory eliminates both:

```
   Socket/pipe IPC (2 copies, 2 syscalls per message):
     proc A buf ──write()──► kernel buffer ──read()──► proc B buf

   Shared memory (0 copies on the data path):
     proc A ─┐                                   ┌─ proc B
             ▼      same physical RAM pages       ▼
     ┌──────────────────────────────────────────────┐
     │   shared region (mapped into BOTH addr spaces) │
     └──────────────────────────────────────────────┘
        A writes here ──────────────► B reads it directly
```

Both processes call `mmap` on the same backing object; the kernel maps the *same physical frames* into each process's virtual address space (at possibly *different* virtual addresses). A store by A is immediately visible to B's load — it's literally the same memory.

That speed comes with a cost the kernel was previously handling for you: **synchronization is now entirely your problem.** A pipe serializes naturally; shared memory is a free-for-all where two processes race on the same bytes unless you coordinate them. This is the entire difficulty of shared-memory IPC.

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 48.2 The POSIX API: shm_open → ftruncate → mmap</h2>

Creating a named shared-memory region is three calls:

```cpp
#include <fcntl.h>
#include <sys/mman.h>
#include <unistd.h>

// 1. Create/open a named shared-memory object (a file in /dev/shm on Linux,
//    a kernel object on macOS). Name must start with '/'.
int fd = ::shm_open("/myregion", O_CREAT | O_RDWR, 0600);

// 2. Size it. A fresh shm object has length 0 — you MUST grow it before mapping.
::ftruncate(fd, sizeof(SharedData));

// 3. Map it into this process's address space. MAP_SHARED makes writes visible
//    to other processes mapping the same object (MAP_PRIVATE would be copy-on-write).
void* addr = ::mmap(nullptr, sizeof(SharedData),
                    PROT_READ | PROT_WRITE, MAP_SHARED, fd, 0);

::close(fd);   // the mapping survives close(); fd no longer needed once mapped
auto* shared = static_cast<SharedData*>(addr);
```

```
   shm_open   ── get a handle (fd) to a named kernel-backed object
   ftruncate  ── set its size (mandatory: new objects are 0 bytes)
   mmap       ── map those pages into your virtual address space (MAP_SHARED)
   ...use...
   munmap     ── unmap from this process
   shm_unlink ── remove the NAME so it can be reclaimed (see lifetime §48.7)
```

The first process creates and sizes it; subsequent processes `shm_open` the same name (without needing `ftruncate`) and `mmap` it. They now share the region.

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 48.3 Named vs Anonymous</h2>

| | **Named** (`shm_open` + name) | **Anonymous** (`mmap(MAP_ANONYMOUS|MAP_SHARED)`) |
|---|---|---|
| Discovery | by string name (`/myregion`) | by inheriting the mapping across `fork()` |
| Use between | **unrelated** processes (no common ancestor) | **related** processes (parent/child after fork) |
| Lifetime | persists until `shm_unlink` (survives process exit!) | freed when all mappers exit/unmap |
| Setup | `shm_open`/`ftruncate`/`mmap` | a single `mmap` before `fork` |

```cpp
// Anonymous: parent maps, then forks — child inherits the SAME mapping.
void* p = ::mmap(nullptr, size, PROT_READ|PROT_WRITE,
                 MAP_SHARED | MAP_ANONYMOUS, -1, 0);
if (::fork() == 0) { /* child sees the same shared region p */ }
```

Use **anonymous** when a parent forks workers (simplest, auto-cleanup). Use **named** when truly independent programs must rendezvous on a region (e.g., a daemon and a separately-launched client). Our build uses **named** so two separately-started processes can connect.

<br><br>

---
---

# PART 2: CROSS-PROCESS SYNCHRONIZATION

---
---

<br>

<h2 style="color: #2980B9;">📘 48.4 Why Synchronization Is Hard Here</h2>

A normal `std::mutex` lives in the heap of *one* process; the other process has its own unrelated `std::mutex` object — they don't coordinate. To synchronize across processes, the synchronization primitive itself must live **inside the shared region** and be initialized to work across address spaces.

Two viable approaches:

1. **Process-shared `pthread_mutex`** — a real mutex, placed in the shared region, with the `PTHREAD_PROCESS_SHARED` attribute set so the kernel knows it's used by multiple processes.
2. **Atomics** — `std::atomic` (or C `<stdatomic.h>`) placed in the shared region, using lock-free protocols. For a single-producer/single-consumer ring buffer this is the clean, fast choice and avoids mutex robustness issues.

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 48.5 Process-Shared pthread_mutex</h2>

To make a mutex usable across processes, set the process-shared attribute *before* initializing it, and place the mutex object inside the shared region:

```cpp
struct SharedData {
    pthread_mutex_t mtx;     // lives in the shared region
    int             value;
};

// Done ONCE by the creating process, after mmap:
pthread_mutexattr_t attr;
::pthread_mutexattr_init(&attr);
::pthread_mutexattr_setpshared(&attr, PTHREAD_PROCESS_SHARED);   // the key line
// (optional but recommended) make it robust: if the lock holder dies, the next
// locker gets EOWNERDEAD instead of deadlocking forever.
::pthread_mutexattr_setrobust(&attr, PTHREAD_MUTEX_ROBUST);
::pthread_mutex_init(&shared->mtx, &attr);
::pthread_mutexattr_destroy(&attr);

// Either process then:
::pthread_mutex_lock(&shared->mtx);
shared->value++;
::pthread_mutex_unlock(&shared->mtx);
```

The **robustness** point is unique to cross-process locks: if a thread holding a normal mutex dies, the program is gone anyway. But if a *process* dies holding a shared mutex, every *other* process deadlocks forever. `PTHREAD_MUTEX_ROBUST` makes the next `lock` return `EOWNERDEAD`, letting survivors recover (call `pthread_mutex_consistent` and proceed, or mark the state corrupt). A plain shared mutex with a process that can crash is a latent deadlock.

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 48.6 Atomics and Memory Ordering Across Processes</h2>

Here's the key insight that surprises people: **memory ordering across processes is the same as across threads, because it's the same CPU and the same physical memory.** The acquire/release semantics you learned for `std::atomic` between threads (Day 4) apply verbatim to atomics in shared memory between processes. There's no extra "cross-process barrier" — the hardware doesn't know or care that the two cores belong to different processes.

So a lock-free SPSC ring buffer works across processes exactly as it does across threads, provided the atomics are placed in the shared region and accessed with correct ordering:

```cpp
struct Ring {
    std::atomic<uint32_t> head;   // consumer advances (next slot to read)
    std::atomic<uint32_t> tail;   // producer advances (next slot to write)
    char data[CAP][SLOT];
};
```

The discipline:

```
   Producer (writes a message):
     1. load tail (relaxed — we own it)
     2. load head (ACQUIRE — see consumer's latest progress; is there room?)
     3. write the payload into data[tail % CAP]
     4. store tail = tail + 1  (RELEASE — payload write must be visible BEFORE
                                 the consumer sees the advanced tail)

   Consumer (reads a message):
     1. load head (relaxed — we own it)
     2. load tail (ACQUIRE — see producer's latest progress; is there data?)
     3. read the payload from data[head % CAP]
     4. store head = head + 1  (RELEASE — payload read done BEFORE producer
                                 sees the slot freed)
```

```
   release/acquire pairing (producer → consumer):

   producer:  data[i] = msg;                  consumer:  if (tail.load(acquire) > head)
              tail.store(t+1, release);  ─────────────►      msg = data[i];   // sees the data
              └ "publish": all prior writes ──────► visible after the acquire load reads t+1
```

`release` on the store + `acquire` on the load form a synchronizes-with edge: everything the producer wrote *before* the release store is guaranteed visible to the consumer *after* its acquire load observes that store. Without the release/acquire, the consumer could see the advanced `tail` but stale payload bytes — a torn read.

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 48.7 Lifetime & Cleanup — the Leak That Survives Your Process</h2>

Named shared memory has **kernel lifetime**, not process lifetime. A region created with `shm_open` persists in `/dev/shm` even after every process that used it exits — until someone calls `shm_unlink` (or the machine reboots). Forgetting to unlink is a real leak that accumulates across runs:

```
   shm_open(O_CREAT)  ── creates the named object (refcount on the NAME)
   mmap / munmap      ── per-process mapping (refcounted separately)
   shm_unlink(name)   ── removes the NAME; the object is freed once the last
                          mmap is gone (like unlink() on an open file)
```

The correct ownership pattern:

- **Mappings** (`mmap`) are per-process: each process `munmap`s on exit.
- **The name** is owned by one party (usually the creator), which calls `shm_unlink` exactly once when the region is no longer needed — typically on clean shutdown.
- `shm_unlink` works like `unlink` on a file: it removes the directory entry immediately, but the underlying object lives until the last mapping is dropped. So unlinking while others are still mapped is safe — they keep working; no *new* process can `shm_open` it afterward.

```bash
# On Linux you can see leaked regions:
ls -l /dev/shm/
# stale "/myregion" sitting there from a crashed run → manually: rm /dev/shm/myregion
```

Defensive habit: the creator should `shm_unlink` an old name *before* `shm_open(O_CREAT)` (clean slate), and `shm_unlink` again on graceful exit.

<br><br>

---
---

# PART 3: BUILD EXERCISE

---
---

<br>

<h2 style="color: #2980B9;">📘 48.8 Exercise: SPSC Ring Buffer in Shared Memory</h2>

Build a lock-free single-producer/single-consumer ring buffer living in a named shared-memory region, with a **producer process** and a **consumer process** (one program, role chosen by argv). Messages are fixed-size slots; the producer enqueues N messages, the consumer dequeues and prints them. Synchronization is via `std::atomic` head/tail with acquire/release ordering — the cross-process version of the classic SPSC queue.

> **Portability note**: Uses POSIX `shm_open`/`ftruncate`/`mmap`/`shm_unlink` (`<sys/mman.h>`, `<fcntl.h>`). Builds on Linux and macOS. Link with `-pthread`; on Linux also `-lrt` is sometimes needed for `shm_*` (glibc ≥ 2.34 folds it into libc). The atomics + memory ordering work identically cross-process because it's the same hardware.

<br>

#### Skeleton — `shm_ring.cpp`

```cpp
// shm_ring.cpp
//   Build:  g++ -std=c++17 -Wall -Wextra -Wpedantic -pthread shm_ring.cpp -o shm_ring
//           (add -lrt on older Linux)
//   Run:    ./shm_ring producer    (in one terminal)
//           ./shm_ring consumer    (in another)
#include <atomic>
#include <cstdint>
#include <cstring>
#include <fcntl.h>
#include <iostream>
#include <sys/mman.h>
#include <unistd.h>

static constexpr const char* SHM_NAME = "/day48_ring";
static constexpr uint32_t CAP   = 1024;   // slots (power of two → cheap modulo)
static constexpr uint32_t SLOT  = 64;     // bytes per slot
static constexpr int      TOTAL = 20;     // messages the producer sends

struct Ring {
    std::atomic<uint32_t> head;           // consumer index (next to read)
    std::atomic<uint32_t> tail;           // producer index (next to write)
    char data[CAP][SLOT];
};

// ── Producer: write TOTAL messages, spin-wait when the ring is full. ──
static int runProducer() {
    int fd = ::shm_open(SHM_NAME, O_CREAT | O_RDWR, 0600);
    if (fd < 0) { perror("shm_open"); return 1; }
    if (::ftruncate(fd, sizeof(Ring)) < 0) { perror("ftruncate"); return 1; }

    void* p = ::mmap(nullptr, sizeof(Ring), PROT_READ | PROT_WRITE, MAP_SHARED, fd, 0);
    if (p == MAP_FAILED) { perror("mmap"); return 1; }
    ::close(fd);
    auto* ring = static_cast<Ring*>(p);

    // Creator initializes the indices (placement-new the atomics).
    new (&ring->head) std::atomic<uint32_t>(0);
    new (&ring->tail) std::atomic<uint32_t>(0);

    std::cout << "[producer] ready; sending " << TOTAL << " messages\n";
    for (int i = 0; i < TOTAL; ++i) {
        uint32_t tail = ring->tail.load(std::memory_order_relaxed);
        // Full when the next tail would collide with head. Leave one slot empty
        // to distinguish full from empty (classic ring-buffer convention).
        while (((tail + 1) % CAP) == (ring->head.load(std::memory_order_acquire) % CAP)) {
            // ring full — wait for the consumer to drain (busy-wait for the demo)
        }
        char msg[SLOT];
        std::snprintf(msg, SLOT, "message #%d", i);
        std::memcpy(ring->data[tail % CAP], msg, SLOT);        // 1) write payload

        ring->tail.store(tail + 1, std::memory_order_release); // 2) publish (RELEASE)
        std::cout << "[producer] sent: " << msg << "\n";
        usleep(50 * 1000);
    }
    // Sentinel so the consumer knows to stop.
    {
        uint32_t tail = ring->tail.load(std::memory_order_relaxed);
        while (((tail + 1) % CAP) == (ring->head.load(std::memory_order_acquire) % CAP)) {}
        std::memcpy(ring->data[tail % CAP], "__DONE__", 9);
        ring->tail.store(tail + 1, std::memory_order_release);
    }
    ::munmap(p, sizeof(Ring));
    std::cout << "[producer] done\n";
    return 0;
}

// ── Consumer: open the existing region, drain until the sentinel. ──
static int runConsumer() {
    int fd = ::shm_open(SHM_NAME, O_RDWR, 0600);   // no O_CREAT: producer made it
    if (fd < 0) { perror("shm_open (start the producer first)"); return 1; }
    void* p = ::mmap(nullptr, sizeof(Ring), PROT_READ | PROT_WRITE, MAP_SHARED, fd, 0);
    if (p == MAP_FAILED) { perror("mmap"); return 1; }
    ::close(fd);
    auto* ring = static_cast<Ring*>(p);

    std::cout << "[consumer] ready; draining\n";
    int count = 0;
    for (;;) {
        uint32_t head = ring->head.load(std::memory_order_relaxed);
        // Empty when head == tail.
        while (head % CAP == (ring->tail.load(std::memory_order_acquire) % CAP)) {
            // ring empty — wait for the producer (busy-wait for the demo)
        }
        char msg[SLOT];
        std::memcpy(msg, ring->data[head % CAP], SLOT);        // read payload (after ACQUIRE)
        ring->head.store(head + 1, std::memory_order_release); // free the slot (RELEASE)

        if (std::strcmp(msg, "__DONE__") == 0) break;
        std::cout << "[consumer] recv: " << msg << "\n";
        ++count;
    }
    ::munmap(p, sizeof(Ring));
    ::shm_unlink(SHM_NAME);    // consumer removes the NAME on clean exit (see §48.7)
    std::cout << "[consumer] done; received " << count << " messages; unlinked region\n";
    return 0;
}

int main(int argc, char** argv) {
    if (argc < 2) {
        std::cerr << "usage: " << argv[0] << " producer|consumer\n";
        return 2;
    }
    std::string role = argv[1];
    if (role == "producer") return runProducer();
    if (role == "consumer") return runConsumer();
    std::cerr << "unknown role: " << role << "\n";
    return 2;
}
```

<br>

#### Compile and run

```bash
g++ -std=c++17 -Wall -Wextra -Wpedantic -pthread shm_ring.cpp -o shm_ring
# (older Linux glibc may also need -lrt)

# Terminal 1 (start the producer first — it creates the region):
./shm_ring producer

# Terminal 2 (within a second or two):
./shm_ring consumer
```

<br>

#### Expected output

```
# producer (terminal 1)
[producer] ready; sending 20 messages
[producer] sent: message #0
[producer] sent: message #1
...
[producer] sent: message #19
[producer] done

# consumer (terminal 2)
[consumer] ready; draining
[consumer] recv: message #0
[consumer] recv: message #1
...
[consumer] recv: message #19
[consumer] done; received 20 messages; unlinked region
```

The messages cross process boundaries with **zero copies through the kernel** — the producer's `memcpy` into the shared slot is directly read by the consumer's `memcpy` out. After a clean run, `/dev/shm/day48_ring` is gone (the consumer unlinked it). If a process crashes mid-run, you may need to `rm /dev/shm/day48_ring` manually — the leak that survives the process.

<br>

#### Bonus Challenges

1. **Prove the ordering matters**: change the producer's `tail.store` to `memory_order_relaxed`. Under ThreadSanitizer (`-fsanitize=thread`, run both roles) or on a weakly-ordered CPU (ARM), watch for torn reads where the consumer sees an advanced tail but stale payload. Restore `release` and confirm it's clean.
2. **Process-shared mutex variant**: re-implement with a `pthread_mutex_t` (`PTHREAD_PROCESS_SHARED` + robust) and a condition variable instead of atomics. Compare throughput and discuss the `EOWNERDEAD` recovery path when you `kill -9` the lock holder.
3. **Blocking instead of busy-wait**: replace the spin loops with a process-shared condition variable (or a named POSIX semaphore `sem_open`) so producer/consumer sleep when full/empty instead of burning CPU. Measure CPU usage before/after.
4. **Variable-length messages**: store `[u32 len][bytes]` in the slots (Day 47 framing) so messages aren't capped at `SLOT`. Handle a message that wraps around the end of the buffer (or forbid wrap by padding to the end).
5. **MPSC / SPMC**: extend to multiple producers (or consumers). Show why the single-atomic head/tail protocol breaks with multiple writers and what you need instead (CAS loop on tail, or per-producer queues).
6. **Crash robustness**: have the consumer detect a stale region from a previous crashed run (e.g., a magic/version + epoch field the producer bumps), `shm_unlink` it, and recreate cleanly — so a crash doesn't poison the next run.

<br><br>

---
---

# PART 4: DEEP DIVE — COMMON QUESTIONS

---
---

<br>

<h2 style="color: #2980B9;">📘 48.9 Q&A</h2>

<br>

#### Q1: "Why is shared memory faster than a pipe or socket?"

A pipe/socket copies data twice (your buffer → kernel buffer on `write`, kernel buffer → peer's buffer on `read`) and crosses the syscall boundary on every message. Shared memory maps the same physical pages into both processes, so a write by one is *directly* readable by the other — zero copies and no syscall on the data path (only the one-time setup `mmap` and the synchronization). For high-throughput, low-latency IPC (databases, message buses, video pipelines) it's the fastest option.

<br>

#### Q2: "What do `shm_open`, `ftruncate`, and `mmap` each do?"

`shm_open` creates or opens a *named* kernel-backed shared object and returns an fd (on Linux it appears under `/dev/shm`). `ftruncate` sets its size — mandatory, because a freshly created object has length 0 and mapping a 0-length object is useless. `mmap(..., MAP_SHARED, fd, 0)` maps those pages into your virtual address space; `MAP_SHARED` is what makes writes visible to other processes mapping the same object. Other processes `shm_open` the same name and `mmap` to join.

<br>

#### Q3: "Named vs anonymous shared memory — when do I use each?"

Anonymous (`mmap` with `MAP_ANONYMOUS|MAP_SHARED` then `fork`) is for *related* processes: the child inherits the mapping, no name needed, and it's auto-freed when all mappers exit — simplest for a parent spawning workers. Named (`shm_open` with a string) is for *unrelated* processes that have no common ancestor and must rendezvous by name — e.g., a daemon and a separately launched client. Named regions also persist beyond process exit until `shm_unlink`, which is both a feature and a leak risk.

<br>

#### Q4: "Why can't I just use a `std::mutex` to coordinate two processes?"

A `std::mutex` is an object in *one* process's address space; the other process has a different, unrelated mutex object — locking one doesn't block the other. To synchronize across processes the primitive must live *inside the shared region* and be initialized as process-shared: a `pthread_mutex_t` with `PTHREAD_PROCESS_SHARED`. Or skip the mutex entirely and use atomics in the shared region (which is what the SPSC ring does).

<br>

#### Q5: "Is memory ordering different across processes than across threads?"

No — and this is the key insight. It's the same CPU cores and the same physical memory, so the hardware's memory-ordering rules are identical. The acquire/release semantics you use for `std::atomic` between threads apply verbatim between processes, with no extra cross-process barrier. A release store followed by an acquire load that observes it establishes the same happens-before edge regardless of whether the two sides are threads or processes. That's why a lock-free SPSC queue ports to shared memory unchanged.

<br>

#### Q6: "Why does the ring buffer need release/acquire and not relaxed?"

The producer does two things: write the payload, then advance `tail`. The consumer observes `tail`, then reads the payload. With `relaxed`, the CPU/compiler may let the consumer see the advanced `tail` *before* the payload write lands — a torn read of stale bytes. The `release` on the producer's `tail.store` publishes all prior writes; the `acquire` on the consumer's `tail.load` makes them visible once it sees the new tail. The pairing guarantees: if you see the index advance, you see the data behind it.

<br>

#### Q7: "What happens if a process crashes while holding the shared mutex?"

With a plain process-shared mutex, the lock is never released and every other process deadlocks forever — a uniquely cross-process failure mode (with a normal in-process mutex, a crash takes the whole program down anyway). The fix is `PTHREAD_MUTEX_ROBUST`: the next thread to `lock` gets `EOWNERDEAD`, signaling the previous owner died mid-critical-section. The survivor must inspect/repair the shared state, call `pthread_mutex_consistent`, and continue — or mark the region unrecoverable. Atomics-based designs sidestep this since there's no held lock.

<br>

#### Q8: "How do I avoid leaking shared memory? What does `shm_unlink` do?"

Named regions have kernel lifetime: they persist after every using process exits, until `shm_unlink` removes the name (or the machine reboots). The pattern: each process `munmap`s its own mapping on exit; *one* owner calls `shm_unlink(name)` exactly once when the region is done (typically on clean shutdown). `shm_unlink` behaves like `unlink` on a file — it removes the name immediately, but the object survives until the last mapping is dropped, so unlinking while peers are still mapped is safe. On Linux, leaked regions show up in `/dev/shm/` and can be `rm`'d manually after a crash.

<br><br>

---

## Reflection Questions

1. Why is shared memory zero-copy while pipes/sockets are two-copy? What does the kernel stop doing for you, and what becomes your responsibility?
2. Walk through `shm_open` → `ftruncate` → `mmap`. Why is `ftruncate` mandatory, and what does `MAP_SHARED` guarantee?
3. When would you choose anonymous shared memory over named, and vice versa?
4. Why doesn't a `std::mutex` work across processes, and what are your two options for cross-process synchronization?
5. Explain why memory ordering across processes is identical to across threads. How do release/acquire prevent a torn read in the ring buffer?
6. What is the lifetime of a named shared-memory region, and what is the cleanup discipline that prevents leaks?

---

## Interview Questions

1. "Why is shared memory the fastest form of IPC? What's the trade-off versus a socket or pipe?"
2. "Walk me through creating a named shared-memory region with the POSIX API."
3. "What's the difference between named and anonymous shared memory? Give a use case for each."
4. "How do you synchronize two processes accessing the same shared region?"
5. "How do you make a `pthread_mutex` work across process boundaries, and why is robustness important there?"
6. "Is memory ordering different across processes than across threads? Justify your answer."
7. "Design a single-producer/single-consumer ring buffer in shared memory. Where do release/acquire go and why?"
8. "A process crashes while holding a process-shared mutex. What happens, and how do you recover?"
9. "What is `shm_unlink`, and what's the leak you get if you forget it?"
10. "How would you extend an SPSC shared-memory queue to multiple producers? What breaks?"

---

**Next**: Day 49 — Weekly Review →
