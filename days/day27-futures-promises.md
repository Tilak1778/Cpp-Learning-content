# Day 27: Futures & Promises

[← Back to Study Plan](../lld-study-plan.md) | [← Day 26](day26-deadlocks-debugging.md)

> **Time**: ~2-3 hours (weekend day)
> **Goal**: Mutexes and condition variables are the *plumbing* of concurrency. Futures and promises are the *result-passing layer* on top: a clean, one-shot channel for moving a value (or an exception) from a producer thread to a consumer thread, with all the synchronization handled for you. Learn `std::future` / `std::promise`, `std::packaged_task`, and the three faces of `std::async` (its `deferred` vs `async` launch policies and the infamous blocking destructor). See how exceptions propagate *through* a future. Then build an **async file processor**: submit file paths, get back futures, collect results — exceptions and all.

---
---

# PART 1: THE FUTURE/PROMISE MODEL

---
---

<br>

<h2 style="color: #2980B9;">📘 27.1 The Core Idea: A One-Shot Value Channel</h2>

A **future** is a handle to a value that *will exist later*. A **promise** is the writing end that *produces* that value. They form a single-use, one-way pipe with a shared synchronization point baked in:

```
   Producer thread                          Consumer thread
   ───────────────                          ───────────────
   std::promise<int> p;                      std::future<int> f = p.get_future();
        │                                          │
        │  ... compute ...                         │  f.get()  ← BLOCKS until value ready
        │                                          │
   p.set_value(42)  ──── shared state ──────►  (unblocks, returns 42)
```

The magic is the **shared state**: a heap-allocated object holding (1) the value-or-exception, (2) a ready flag, and (3) the synchronization (a condition variable internally). The promise writes into it; the future reads from it; the standard handles the wait. You never touch a mutex or condition variable yourself.

| | Promise | Future |
|---|---------|--------|
| Role | the *producer* / writing end | the *consumer* / reading end |
| Key op | `set_value(v)` / `set_exception(e)` | `get()` (blocks until ready) |
| Count | exactly one | exactly one (or `shared_future` for many) |
| Reusable? | No — single value, then done | No — `get()` may be called once |

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 27.2 std::future — The Consumer End</h2>

```cpp
std::future<int> f = /* ... */;

int v = f.get();          // BLOCKS until the value is set; returns it (or rethrows exception).
                          // Calling get() a SECOND time is UB — the value was moved out.

f.wait();                 // block until ready, but don't retrieve the value yet
auto status = f.wait_for(std::chrono::milliseconds(100));   // timed, non-consuming
// status ∈ { ready, timeout, deferred }

if (f.valid()) { /* this future still refers to a shared state */ }
```

Two non-obvious rules:

- **`get()` is one-shot.** It *moves* the result out of the shared state. A second `get()` is undefined behavior. If you need multiple readers, use `std::shared_future` (its `get()` returns a `const&` and may be called repeatedly by copies).
- **`get()` rethrows.** If the producer stored an exception, `get()` throws it in the *consumer's* thread (covered in §27.6).

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 27.3 std::promise — The Producer End</h2>

`std::promise` is the lowest-level producer. You set the value (or exception) by hand — useful when the result comes from somewhere `async`/`packaged_task` can't express (a callback, an event loop, a custom thread).

```cpp
std::promise<int> p;
std::future<int> f = p.get_future();    // get the paired future ONCE

std::thread producer([&p]{
    try {
        int result = expensiveComputation();
        p.set_value(result);            // hand the value across the channel
    } catch (...) {
        p.set_exception(std::current_exception());   // or hand the exception across
    }
});

int v = f.get();    // consumer blocks here, then gets value or rethrows
producer.join();
```

#### The broken-promise trap

If a `promise` is destroyed *without* ever setting a value or exception, the shared state is marked with a `std::future_error` of code `broken_promise`, and the waiting future's `get()` throws it. This is the standard's safety net against "consumer waits forever for a value that will never come" — but it's a sign of a logic bug (a producer that forgot to deliver).

<br><br>

---
---

# PART 2: PACKAGED_TASK & ASYNC

---
---

<br>

<h2 style="color: #2980B9;">📘 27.4 std::packaged_task — Wrap a Callable's Result in a Future</h2>

A `std::packaged_task<Sig>` wraps a callable so that *invoking it* stores the return value (or exception) into an associated future. It's the bridge between "a callable" and "a future" — the natural unit of work for a **thread pool**.

```cpp
std::packaged_task<int(int,int)> task([](int a, int b){ return a + b; });
std::future<int> f = task.get_future();

std::thread t(std::move(task), 2, 3);   // packaged_task is move-only; run it on some thread
t.join();

std::cout << f.get();   // 5
```

The thread-pool pattern: producers push `packaged_task`s onto a queue (Day 23's bounded queue!), worker threads pop and invoke them, and the submitter holds the future to retrieve the result. `packaged_task` is exactly the "task" abstraction a pool schedules.

<br>

#### The three result-producers compared

| | `promise` | `packaged_task` | `async` |
|---|-----------|-----------------|---------|
| You set the result | manually (`set_value`) | automatically (return value of the callable) | automatically |
| You control where it runs | yes (any thread) | yes (you invoke it where you like) | the policy decides |
| Best for | results from callbacks/event loops | tasks scheduled by *your* executor (thread pool) | fire-and-collect convenience |

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 27.5 std::async — Launch Policies</h2>

`std::async` is the high-level convenience: hand it a callable, get back a future. But its *launch policy* changes everything.

```cpp
// Explicit async: runs on a (possibly new) thread NOW, concurrently.
std::future<int> f1 = std::async(std::launch::async, work, 1, 2);

// Deferred: does NOT run yet — runs LAZILY on the calling thread when you call f.get()/wait().
std::future<int> f2 = std::async(std::launch::deferred, work, 1, 2);

// Default (async | deferred): the IMPLEMENTATION CHOOSES. Dangerous — see below.
std::future<int> f3 = std::async(work, 1, 2);
```

| Policy | When it runs | On which thread |
|--------|--------------|-----------------|
| `launch::async` | immediately, concurrently | a separate thread |
| `launch::deferred` | lazily, on first `get()`/`wait()` | the *calling* thread (no concurrency!) |
| default (both) | implementation's choice | unspecified |

<br>

#### Why the default policy is a trap

With the default, the implementation may pick `deferred` — meaning your "parallel" tasks never actually run on other threads, and worse, a deferred task **never runs at all** if you never call `get()`. Code that "works" in testing (where you happen to call `get()`) can silently serialize or skip work in production. **Always pass an explicit policy** — almost always `std::launch::async` when you want concurrency.

<br>

#### The infamous blocking destructor

A future returned from `std::async(std::launch::async, ...)` has a special property: **its destructor blocks until the task finishes.** This surprises everyone:

```cpp
{
    std::async(std::launch::async, longRunningTask);   // temporary future, destroyed at ';'
    // ↑ this line BLOCKS until longRunningTask completes — it is NOT fire-and-forget!
    doOtherWork();   // does not start until longRunningTask is done
}
```

The temporary future's destructor joins the task. So `std::async(...)` with no captured future is *synchronous*, not background. To actually run concurrently, **keep the future alive** in a named variable for as long as you want the task running. This blocking-destructor behavior is unique to `async`-launched futures; `promise`/`packaged_task` futures do **not** block on destruction.

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 27.6 Exception Propagation Through Futures</h2>

This is the feature that makes futures so much nicer than raw threads. If a task throws, the exception is **captured** into the shared state and **rethrown** when the consumer calls `get()` — crossing the thread boundary cleanly.

```cpp
std::future<int> f = std::async(std::launch::async, []() -> int {
    throw std::runtime_error("task failed");
});

try {
    int v = f.get();          // the runtime_error is rethrown HERE, in the consumer's thread
} catch (const std::runtime_error& e) {
    std::cout << "caught: " << e.what() << "\n";   // "caught: task failed"
}
```

Compare with a raw `std::thread`: an exception escaping the thread function calls `std::terminate()` and kills the process — there's no built-in way to ship it back. Futures fix this by storing `std::current_exception()` (a `std::exception_ptr`) in the shared state and rethrowing it at `get()`.

```
Worker thread                         Consumer thread
─────────────                         ───────────────
throw runtime_error  ──► stored as     f.get()  ──► exception_ptr rethrown here
                         exception_ptr               → ordinary try/catch works
```

For manual promises, you do this yourself: `catch(...) { p.set_exception(std::current_exception()); }`. The error travels through the same one-shot channel as a value would.

<br><br>

---
---

# PART 3: BUILD EXERCISE

---
---

<br>

<h2 style="color: #2980B9;">📘 27.7 Exercise: Async File Processor</h2>

Build a processor that accepts file paths, processes each on a worker (counting lines/words, or throwing on a missing file), and returns a `std::future` per submission. The driver collects all futures and prints results — including catching the exceptions that propagate through futures for missing files.

<br>

#### Skeleton (header)

```cpp
// async_processor.h
#pragma once
#include <future>
#include <string>
#include <vector>
#include <fstream>
#include <sstream>
#include <stdexcept>
#include <cstddef>

struct FileStats {
    std::string path;
    std::size_t lines = 0;
    std::size_t words = 0;
    std::size_t bytes = 0;
};

// Does the real work. Throws std::runtime_error if the file can't be opened —
// this exception will propagate through the future to the submitter's get().
inline FileStats processFile(const std::string& path) {
    std::ifstream in(path, std::ios::binary);
    if (!in) throw std::runtime_error("cannot open file: " + path);

    FileStats s;
    s.path = path;
    std::string line;
    while (std::getline(in, line)) {
        ++s.lines;
        s.bytes += line.size() + 1;        // +1 approximates the newline
        std::istringstream iss(line);
        std::string w;
        while (iss >> w) ++s.words;
    }
    return s;
}

class AsyncFileProcessor {
public:
    // Submit a path; returns a future that will hold the stats (or rethrow).
    // Explicit launch::async so each file is processed concurrently.
    std::future<FileStats> submit(const std::string& path) {
        return std::async(std::launch::async, processFile, path);
    }

    // Convenience: submit a batch, return all futures.
    std::vector<std::future<FileStats>> submitAll(const std::vector<std::string>& paths) {
        std::vector<std::future<FileStats>> futures;
        futures.reserve(paths.size());
        for (const auto& p : paths)
            futures.push_back(submit(p));   // all launched concurrently
        return futures;
    }
};
```

<br>

#### Test driver

```cpp
// main.cpp
#include "async_processor.h"
#include <iostream>
#include <fstream>
#include <cassert>

// Create a sample file so the test is self-contained.
static void writeSample(const std::string& path, const std::string& contents) {
    std::ofstream out(path);
    out << contents;
}

int main() {
    std::cout << "=== Async file processor ===\n";

    writeSample("sample_a.txt", "hello world\nfoo bar baz\n");          // 2 lines, 5 words
    writeSample("sample_b.txt", "one two\nthree\nfour five six\n");      // 3 lines, 6 words

    AsyncFileProcessor proc;

    std::vector<std::string> paths = {
        "sample_a.txt",
        "sample_b.txt",
        "does_not_exist.txt",     // will throw → exception propagates through the future
    };

    auto futures = proc.submitAll(paths);   // all three launched concurrently

    std::size_t okCount = 0, errCount = 0;
    std::size_t totalLines = 0, totalWords = 0;

    for (std::size_t i = 0; i < futures.size(); ++i) {
        try {
            FileStats s = futures[i].get();   // blocks; rethrows if the task threw
            std::cout << "  [ok ] " << s.path
                      << "  lines=" << s.lines
                      << " words=" << s.words
                      << " bytes=" << s.bytes << "\n";
            totalLines += s.lines;
            totalWords += s.words;
            ++okCount;
        } catch (const std::exception& e) {
            std::cout << "  [err] " << paths[i] << "  -> " << e.what() << "\n";
            ++errCount;
        }
    }

    std::cout << "  processed ok=" << okCount << " errors=" << errCount
              << " totalLines=" << totalLines << " totalWords=" << totalWords << "\n";

    assert(okCount == 2);
    assert(errCount == 1);            // the missing file's exception was caught via get()
    assert(totalLines == 5);          // 2 + 3
    assert(totalWords == 11);         // 5 + 6

    std::cout << "All assertions passed.\n";
    return 0;
}
```

<br>

#### Compile and run

```bash
g++ -std=c++17 -Wall -Wextra -Wpedantic -O2 -pthread \
    -o day27 main.cpp && ./day27
```

#### Under ThreadSanitizer

```bash
g++ -std=c++17 -Wall -Wextra -Wpedantic -g -fsanitize=thread -pthread \
    -o day27_tsan main.cpp && ./day27_tsan
```

The key lesson is in the third path: `processFile` throws for the missing file, the exception is stored in the future's shared state, and `futures[i].get()` rethrows it in the main thread, where the ordinary `try/catch` handles it. No `terminate`, no special machinery.

<br>

#### Expected output

```
=== Async file processor ===
  [ok ] sample_a.txt  lines=2 words=5 bytes=...
  [ok ] sample_b.txt  lines=3 words=6 bytes=...
  [err] does_not_exist.txt  -> cannot open file: does_not_exist.txt
  processed ok=2 errors=1 totalLines=5 totalWords=11
All assertions passed.
```

(Result *order* is deterministic here because we `get()` the futures in submission order, even though processing ran concurrently.)

<br>

#### Bonus Challenges

1. **Replace `async` with a thread pool.** Build a fixed pool of N worker threads pulling `std::packaged_task<FileStats()>` off a Day-23 bounded queue. `submit` enqueues a task and returns its future. Compare against `std::async` (which may spawn an unbounded number of threads). Why does the pool scale better for thousands of files?

2. **Prove the blocking-destructor trap.** Call `proc.submit(path)` and *discard* the returned future (don't store it). Add timing prints inside `processFile`. Show that the submitting loop blocks at each discarded future's destruction — i.e., it ran *serially*, not concurrently. Then keep the futures and show real concurrency.

3. **`shared_future` fan-out.** Convert a future to `std::shared_future` and have several threads each call `get()` on a copy to read the same result. Explain why a plain `future` can't be read twice.

4. **Manual promise version.** Re-implement `submit` using `std::promise` + a raw `std::thread`: the thread computes, then `set_value`/`set_exception`. Show it behaves identically to the `async` version, and note that the promise's future does *not* block on destruction.

5. **`wait_for` progress / timeout.** Poll the futures with `wait_for(0ms)` to print which files have finished, and add a per-file timeout: if a file isn't done in N ms, report it as "slow" (don't cancel — there's no standard cancellation; discuss why).

6. **Default-policy footgun.** Change `submit` to call `std::async(processFile, path)` *without* an explicit policy. On a platform that picks `deferred`, demonstrate that the files are processed serially on the main thread at `get()` time (no concurrency) — and that a never-`get()`-ed task never runs at all. Then restore `launch::async` and explain the fix.

<br><br>

---
---

# PART 4: DEEP DIVE — COMMON QUESTIONS

---
---

<br>

<h2 style="color: #2980B9;">📘 27.8 Q&A</h2>

<br>

#### Q1: "What's the difference between a future and a promise?"

They're the two ends of a one-shot value channel sharing a heap-allocated *shared state*. The **promise** is the producer end — you call `set_value`/`set_exception` to deliver the result. The **future** is the consumer end — you call `get()` to retrieve it (blocking until ready). One promise, one future, one value. The standard handles all the synchronization between them.

<br>

#### Q2: "Why does `std::async` with the default policy surprise people?"

Because the default is `launch::async | launch::deferred`, letting the implementation choose `deferred` — which runs the task *lazily on the calling thread* at `get()` time, giving you zero concurrency, and runs *not at all* if you never call `get()`. Code that looks parallel can silently serialize or skip work. Always pass an explicit policy; use `launch::async` for real concurrency.

<br>

#### Q3: "Why does a future from `std::async` block in its destructor?"

`std::async(launch::async, ...)` returns a future whose destructor *waits* for the launched task to complete (effectively joining it). This prevents a task from running on freed resources after its launcher disappears, but it means a discarded/temporary future makes `async` synchronous, not fire-and-forget. Keep the future alive in a variable to actually overlap work. Futures from `promise`/`packaged_task` do **not** block on destruction.

<br>

#### Q4: "How do exceptions cross from a worker thread to the caller?"

If a task throws, the exception is captured as a `std::exception_ptr` (via `std::current_exception()`) and stored in the shared state instead of a value. When the consumer calls `get()`, that exception is rethrown in the consumer's thread, so an ordinary `try/catch` around `get()` handles it. With a raw `std::thread`, an escaping exception instead calls `std::terminate()` — futures are how you get exception safety across threads.

<br>

#### Q5: "When do I use `promise` vs `packaged_task` vs `async`?"

`std::async` for simple "run this and give me a future" convenience. `std::packaged_task` when *you* control scheduling — it wraps a callable into a task you push onto your own thread pool's queue. `std::promise` when the result arrives from somewhere those two can't express — a callback, an event loop, or hand-rolled producer logic where you set the value manually.

<br>

#### Q6: "Why can I call `get()` only once on a `std::future`?"

`get()` *moves* the result out of the shared state, leaving the future invalid; a second call is undefined behavior. This single-consumer model keeps `future` cheap and unambiguous. If multiple threads must read the same result, convert to `std::shared_future`, whose `get()` returns a reference and may be called repeatedly across copies.

<br>

#### Q7: "What is a 'broken promise'?"

If a `std::promise` is destroyed without ever setting a value or exception, the shared state is completed with a `std::future_error` of code `broken_promise`, and the waiting future's `get()` throws it — rather than hanging forever. It's a safety net that surfaces a logic bug: a producer that was supposed to deliver a result but didn't.

<br>

#### Q8: "Can I cancel a future or a running task?"

No — the standard library has **no cancellation** for `future`/`async`/`packaged_task`. Once a task is running, you can only wait for it. If you need cancellation, you build it cooperatively: the task periodically checks a `std::atomic<bool>` cancel flag (or a C++20 `std::stop_token`) and returns early when set. This is why `std::jthread`'s `stop_token` exists, and why robust systems pass a cancellation token into long-running work.

<br><br>

---

## Reflection Questions

1. Describe the shared state. What three things does it hold, and who reads/writes each?
2. Why is `std::async`'s default launch policy dangerous? What should you always do instead?
3. Explain the blocking-destructor behavior of an `async`-launched future. How does it turn "background" work synchronous?
4. How does an exception travel from a worker thread to the caller through a future? Contrast with a raw `std::thread`.
5. When would you choose `promise` over `packaged_task` over `async`?
6. Why can a `std::future` be `get()`-ten only once, and when do you reach for `shared_future`?

---

## Interview Questions

1. "Explain `std::future` and `std::promise`. How do they communicate?"
2. "What are the launch policies of `std::async`? Why is the default a footgun?"
3. "Why does a future returned by `std::async` block in its destructor? What's the consequence?"
4. "How does an exception thrown in a task reach the thread that holds the future?"
5. "Compare `std::promise`, `std::packaged_task`, and `std::async`. When use each?"
6. "Why can you call `get()` only once on a `std::future`? What's `std::shared_future` for?"
7. "Build a thread pool where `submit` returns a `std::future` for each task."
8. "What is a broken promise, and when does it happen?"
9. "How would you cancel a running async task, given the standard has no cancellation?"
10. "Implement an async file/URL processor: submit work, collect results, and handle per-item failures via futures."

---

**Next**: Day 28 — Weekly Review →
