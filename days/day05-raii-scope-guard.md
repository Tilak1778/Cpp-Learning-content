# Day 5: RAII — ScopeGuard & FileHandle

[← Back to Study Plan](../lld-study-plan.md) | [← Day 4](day04-review-atomic-refcount.md)

> **Time**: ~2 hours
> **Goal**: Understand RAII as a foundational C++ idiom. Build a general-purpose `ScopeGuard` for arbitrary cleanup, and an RAII `FileHandle` wrapper for POSIX file descriptors.

---
---

# PART 1: THEORY (30 min)

---

<br>

<h2 style="color: #2980B9;">📘 5.1 What Is RAII?</h2>

**RAII** stands for **Resource Acquisition Is Initialization**. The name is confusing — what it really means is:

> **Tie a resource's lifetime to an object's lifetime.** Acquire in the constructor, release in the destructor.

When the object goes out of scope — whether normally, via `return`, or via exception — the destructor runs and the resource is freed. **No manual cleanup needed.**

```cpp
// WITHOUT RAII — manual cleanup, error-prone
void process() {
    FILE* f = fopen("data.txt", "r");
    if (!f) return;                     // forgot to close? no, file wasn't opened

    int* buf = new int[1024];

    if (badCondition()) {
        fclose(f);                      // must remember to close
        delete[] buf;                   // must remember to free
        return;
    }

    doWork(f, buf);

    if (anotherError()) {
        fclose(f);                      // duplicated cleanup
        delete[] buf;                   // duplicated cleanup
        throw std::runtime_error("fail");  // WAIT — this leaks if we forget above!
    }

    fclose(f);                          // 3rd copy of cleanup
    delete[] buf;
}
```

Every early return, every exception path needs manual cleanup. Miss one → leak.

```cpp
// WITH RAII — automatic cleanup
void process() {
    auto f = FileHandle("data.txt", O_RDONLY);   // constructor opens
    auto buf = std::make_unique<int[]>(1024);     // constructor allocates

    if (badCondition()) return;          // f and buf destructors clean up
    doWork(f.fd(), buf.get());
    if (anotherError()) throw std::runtime_error("fail");  // still cleaned up!
}   // f closes, buf deletes — even on exception
```

**Every resource in C++ should be wrapped in an RAII object.** This is the single most important C++ idiom.

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 5.2 RAII You Already Know</h2>

You've been using RAII all week without calling it that:

| RAII Wrapper | Resource | Acquire (ctor) | Release (dtor) |
|-------------|----------|-----------------|-----------------|
| `std::string` | Heap-allocated char buffer | `new char[]` | `delete[]` |
| `std::vector` | Heap-allocated array | `new T[]` | `delete[]` |
| `std::unique_ptr` | Heap object | stores raw pointer | `delete` / custom deleter |
| `std::shared_ptr` | Heap object + control block | allocates control block | `delete` when last owner |
| `std::lock_guard` | Mutex lock | `mutex.lock()` | `mutex.unlock()` |
| `std::fstream` | File handle | `open()` | `close()` |
| Your `String` (Day 1) | `char*` | `new char[]` | `delete[]` |

The pattern is always the same: **constructor acquires, destructor releases, no exceptions possible in cleanup.**

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 5.3 The ScopeGuard Pattern</h2>

Sometimes you need RAII cleanup but don't want to write a whole class. A **ScopeGuard** is a generic RAII wrapper that takes **any callable** and runs it in the destructor:

```cpp
auto guard = ScopeGuard([] {
    std::cout << "Cleanup runs no matter what!\n";
});
```

Real-world uses:

```cpp
void transferMoney(Account& from, Account& to, int amount) {
    from.withdraw(amount);

    // If deposit fails, we need to rollback the withdrawal
    auto rollback = ScopeGuard([&] { from.deposit(amount); });

    to.deposit(amount);     // might throw!

    rollback.dismiss();     // success — don't rollback
}
```

If `to.deposit()` throws, the ScopeGuard's destructor runs `from.deposit(amount)` to undo the withdrawal. If everything succeeds, `dismiss()` tells the guard not to run cleanup.

This is a form of **transactional programming** — you set up the "undo" immediately after each operation, then dismiss it on success.

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 5.4 Designing ScopeGuard — Key Decisions</h2>

<br>

#### What type is the callable?

Option 1: **Template** — zero overhead, stores any lambda/functor inline

```cpp
template <typename F>
class ScopeGuard {
    F func_;
    bool active_;
};
```

Option 2: **`std::function`** — type-erased, flexible, but has heap allocation overhead

```cpp
class ScopeGuard {
    std::function<void()> func_;
    bool active_;
};
```

For a ScopeGuard, **Option 1 (template) is preferred**. The guard is typically local and short-lived, so zero overhead matters. You'll implement `std::function`-like type erasure on Day 51.

<br>

#### Should it be copyable?

**No.** Copying a ScopeGuard would mean the cleanup runs twice — double free, double close, double rollback. It should be **move-only** (like `unique_ptr`):

```cpp
ScopeGuard(const ScopeGuard&) = delete;
ScopeGuard& operator=(const ScopeGuard&) = delete;
ScopeGuard(ScopeGuard&& other) noexcept;
```

<br>

#### The `dismiss()` method

Sets a flag so the destructor **skips** cleanup. Used when the "happy path" succeeds and you no longer need the rollback:

```cpp
void dismiss() noexcept { active_ = false; }
```

<br>

#### Should the destructor be `noexcept`?

**Yes, always.** Destructors are implicitly `noexcept` in C++11+. If your cleanup function throws, `std::terminate` is called. This is correct — you can't meaningfully handle an exception during stack unwinding. Make sure your cleanup lambdas don't throw.

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 5.5 POSIX File Descriptors — Why They Need RAII</h2>

On UNIX/macOS/Linux, files, sockets, pipes, and devices are all accessed through **file descriptors** — small integers returned by system calls:

```cpp
#include <fcntl.h>
#include <unistd.h>

int fd = open("data.txt", O_RDONLY);   // returns fd (e.g., 3)
read(fd, buffer, size);                 // use the fd
close(fd);                              // must manually close
```

Problems without RAII:
- Forget `close()` → **file descriptor leak** (system has a finite limit, typically 1024)
- Close twice → undefined behavior
- Exception between open and close → leak
- Return early → leak

An RAII wrapper makes this foolproof:

```cpp
{
    FileHandle f("data.txt", O_RDONLY);
    read(f.fd(), buffer, size);
}   // automatically closed here — no matter what
```

<br>

#### The "invalid fd" sentinel

`open()` returns `-1` on failure. So our RAII wrapper uses `-1` as the "empty" state (like `nullptr` for pointers):

```cpp
class FileHandle {
    int fd_ = -1;   // -1 means "no file open"
};
```

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 5.6 FileHandle Design — Move-Only Resource Wrapper</h2>

A file descriptor is an **exclusive resource** — exactly one owner should close it. This is the same pattern as `unique_ptr`:

| | `unique_ptr` | `FileHandle` |
|---|---|---|
| Resource | heap pointer | file descriptor (int) |
| Acquire | `new T` | `open()` |
| Release | `delete` | `close()` |
| Empty state | `nullptr` | `-1` |
| Copyable? | No | No |
| Movable? | Yes | Yes |
| Move semantics | Steal pointer, null source | Steal fd, set source to -1 |

The implementation follows the same Rule of 5 pattern you learned on Day 1:

```
Constructor:      open the file
Destructor:       close if fd_ != -1
Copy:             deleted (no two owners)
Move constructor: steal fd, set source to -1
Move assignment:  close current, steal from source
```

<br><br>

---
---

# PART 2: BUILD EXERCISE (60 min)

---

<br>

## Exercise A: Implement `ScopeGuard`

<br>

### Starter skeleton:

```cpp
#include <iostream>
#include <utility>

template <typename F>
class ScopeGuard {
private:
    F func_;
    bool active_;

public:
    // Constructor — takes a callable
    // YOUR IMPLEMENTATION HERE

    // Destructor — calls func_ if active_
    // YOUR IMPLEMENTATION HERE

    // No copying
    // YOUR IMPLEMENTATION HERE

    // Move constructor — transfer ownership, deactivate source
    // YOUR IMPLEMENTATION HERE

    // Dismiss — prevent cleanup from running
    void dismiss() noexcept { active_ = false; }
};

// Helper function to deduce template parameter (C++17: CTAD makes this optional)
template <typename F>
ScopeGuard<F> make_scope_guard(F&& f) {
    return ScopeGuard<F>(std::forward<F>(f));
}
```

<br>

### Test driver:

```cpp
#include <stdexcept>

void test_basic() {
    std::cout << "=== 1. Basic — cleanup on scope exit ===\n";
    {
        auto guard = make_scope_guard([] {
            std::cout << "  Cleanup executed!\n";
        });
        std::cout << "  Inside scope\n";
    }   // "Cleanup executed!" prints here
}

void test_dismiss() {
    std::cout << "\n=== 2. Dismiss — cleanup skipped ===\n";
    {
        auto guard = make_scope_guard([] {
            std::cout << "  This should NOT print\n";
        });
        std::cout << "  Dismissing guard\n";
        guard.dismiss();
    }   // nothing prints — dismissed
    std::cout << "  Guard was dismissed successfully\n";
}

void test_exception() {
    std::cout << "\n=== 3. Exception — cleanup still runs ===\n";
    try {
        auto guard = make_scope_guard([] {
            std::cout << "  Cleanup on exception!\n";
        });
        std::cout << "  About to throw\n";
        throw std::runtime_error("boom");
    } catch (const std::exception& e) {
        std::cout << "  Caught: " << e.what() << "\n";
    }
}

void test_transaction() {
    std::cout << "\n=== 4. Transaction rollback pattern ===\n";
    int balance = 100;

    auto withdraw = [&](int amount) {
        balance -= amount;
        std::cout << "  Withdrew " << amount << ", balance: " << balance << "\n";

        auto rollback = make_scope_guard([&] {
            balance += amount;
            std::cout << "  ROLLBACK: balance restored to " << balance << "\n";
        });

        // Simulate failure
        if (amount > 50) {
            std::cout << "  Transfer failed!\n";
            return;   // rollback runs
        }

        rollback.dismiss();   // success — no rollback
        std::cout << "  Transfer succeeded!\n";
    };

    withdraw(30);     // should succeed, no rollback
    withdraw(80);     // should fail, rollback
    std::cout << "  Final balance: " << balance << "\n";   // should be 70
}

int main() {
    test_basic();
    test_dismiss();
    test_exception();
    test_transaction();
    return 0;
}
```

<br>

### Expected output:

```
=== 1. Basic — cleanup on scope exit ===
  Inside scope
  Cleanup executed!

=== 2. Dismiss — cleanup skipped ===
  Dismissing guard
  Guard was dismissed successfully

=== 3. Exception — cleanup still runs ===
  About to throw
  Cleanup on exception!
  Caught: boom

=== 4. Transaction rollback pattern ===
  Withdrew 30, balance: 70
  Transfer succeeded!
  Withdrew 80, balance: -10
  Transfer failed!
  ROLLBACK: balance restored to 70
  Final balance: 70
```

<br><br>

---

<br>

## Exercise B: Implement `FileHandle`

<br>

### Starter skeleton:

```cpp
#include <iostream>
#include <fcntl.h>
#include <unistd.h>
#include <cerrno>
#include <cstring>
#include <utility>

class FileHandle {
private:
    int fd_;

public:
    // Constructor — open file (or take existing fd)
    // YOUR IMPLEMENTATION HERE

    // Constructor from raw fd
    explicit FileHandle(int fd = -1) : fd_(fd) {}

    // Destructor — close if valid
    // YOUR IMPLEMENTATION HERE

    // No copy
    // YOUR IMPLEMENTATION HERE

    // Move constructor — steal fd, set source to -1
    // YOUR IMPLEMENTATION HERE

    // Move assignment — close current, steal from source
    // YOUR IMPLEMENTATION HERE

    // Access the raw fd
    int fd() const noexcept { return fd_; }

    // Check if valid
    bool is_valid() const noexcept { return fd_ >= 0; }
    explicit operator bool() const noexcept { return is_valid(); }

    // Release ownership (like unique_ptr::release)
    int release() noexcept {
        int tmp = fd_;
        fd_ = -1;
        return tmp;
    }

    // Close current and optionally take new fd
    void reset(int new_fd = -1) noexcept {
        int old = fd_;
        fd_ = new_fd;
        if (old >= 0) ::close(old);
    }

    // Swap
    friend void swap(FileHandle& a, FileHandle& b) noexcept {
        using std::swap;
        swap(a.fd_, b.fd_);
    }
};
```

<br>

### Test driver:

```cpp
#include <cassert>

int main() {
    std::cout << "=== 1. Open and auto-close ===\n";
    {
        FileHandle f(::open("/dev/null", O_RDONLY));
        std::cout << "  fd = " << f.fd() << "\n";
        assert(f.is_valid());
        std::cout << "  is_valid: " << f.is_valid() << "\n";
    }   // closed here
    std::cout << "  (file closed)\n";

    std::cout << "\n=== 2. Default (empty) state ===\n";
    {
        FileHandle empty;
        assert(!empty.is_valid());
        assert(empty.fd() == -1);
        std::cout << "  empty fd: " << empty.fd() << "\n";
    }

    std::cout << "\n=== 3. Move construction ===\n";
    {
        FileHandle a(::open("/dev/null", O_RDONLY));
        int original_fd = a.fd();
        FileHandle b = std::move(a);
        assert(a.fd() == -1);           // source is empty
        assert(b.fd() == original_fd);  // destination has the fd
        std::cout << "  a.fd() = " << a.fd() << " (moved from)\n";
        std::cout << "  b.fd() = " << b.fd() << " (owns it now)\n";
    }

    std::cout << "\n=== 4. Move assignment ===\n";
    {
        FileHandle a(::open("/dev/null", O_RDONLY));
        FileHandle b(::open("/dev/null", O_WRONLY));
        std::cout << "  before: a=" << a.fd() << " b=" << b.fd() << "\n";
        a = std::move(b);  // a's old fd closed, a now has b's fd
        std::cout << "  after:  a=" << a.fd() << " b=" << b.fd() << "\n";
        assert(b.fd() == -1);
    }

    std::cout << "\n=== 5. release() ===\n";
    {
        FileHandle f(::open("/dev/null", O_RDONLY));
        int raw = f.release();
        assert(f.fd() == -1);
        assert(raw >= 0);
        std::cout << "  released fd: " << raw << "\n";
        ::close(raw);   // manual cleanup since we released
    }

    std::cout << "\n=== 6. reset() ===\n";
    {
        FileHandle f(::open("/dev/null", O_RDONLY));
        std::cout << "  before reset: " << f.fd() << "\n";
        f.reset(::open("/dev/null", O_WRONLY));  // closes old, takes new
        std::cout << "  after reset:  " << f.fd() << "\n";
        f.reset();  // closes, becomes -1
        std::cout << "  after reset(): " << f.fd() << "\n";
    }

    std::cout << "\n=== 7. Copy is deleted (uncomment to verify) ===\n";
    // FileHandle a(::open("/dev/null", O_RDONLY));
    // FileHandle b = a;         // should NOT compile
    // FileHandle c;
    // c = a;                    // should NOT compile
    std::cout << "  (copy lines commented out — uncomment to verify compile error)\n";

    std::cout << "\n=== 8. operator bool ===\n";
    {
        FileHandle valid(::open("/dev/null", O_RDONLY));
        FileHandle invalid;
        std::cout << "  valid: " << (valid ? "yes" : "no") << "\n";
        std::cout << "  invalid: " << (invalid ? "yes" : "no") << "\n";
    }

    std::cout << "\n=== Done ===\n";
    return 0;
}
```

<br>

### Compile and run:

```bash
g++ -std=c++17 -Wall -Wextra -Wpedantic -fsanitize=address -o day05 day05_raii.cpp && ./day05
```

<br><br>

---
---

# PART 3: DEEP DIVE CHALLENGES (20 min)

---

<br>

## Challenge 1: `ScopeGuard` with `ON_SCOPE_EXIT` macro

Real codebases use a macro for convenience:

```cpp
#define CONCAT_IMPL(a, b) a##b
#define CONCAT(a, b) CONCAT_IMPL(a, b)
#define ON_SCOPE_EXIT auto CONCAT(scope_guard_, __LINE__) = make_scope_guard

// Usage:
ON_SCOPE_EXIT([&] { cleanup(); });
```

The `__LINE__` trick creates a unique variable name per line so you don't have to name each guard.

<br>

## Challenge 2: `ScopeGuard` that only runs on exception

C++17 has `std::uncaught_exceptions()` (note the plural). You can check if the guard's destructor is running due to normal scope exit or stack unwinding:

```cpp
~ScopeGuard() {
    if (active_ && std::uncaught_exceptions() > uncaught_on_creation_) {
        func_();   // only run on exception
    }
}
```

Implement `ScopeFailGuard` that only runs cleanup when an exception is in flight.

<br>

## Challenge 3: FileHandle with read/write helpers

Add convenience methods:

```cpp
ssize_t read(void* buf, size_t count);
ssize_t write(const void* buf, size_t count);
off_t seek(off_t offset, int whence = SEEK_SET);
```

Handle partial reads/writes (loop until all bytes are transferred or error).

<br>

## Challenge 4: Generic `UniqueResource<T, Deleter>`

Generalize `FileHandle` to wrap **any** resource type with any cleanup function:

```cpp
template <typename T, typename Deleter>
class UniqueResource {
    T resource_;
    Deleter deleter_;
    bool engaged_;
};

// Usage:
auto handle = UniqueResource(CreateWindow(...), &DestroyWindow);
auto mem = UniqueResource(malloc(1024), &free);
```

This is what `std::experimental::unique_resource` (Library Fundamentals TS v3) does.

<br><br>

---
---

# PART 4: DEEP DIVE Q&A

---

<br>

<h2 style="color: #2980B9;">📘 Q1: How is ScopeGuard different from <code>unique_ptr</code> with a custom deleter?</h2>

They're similar — both run cleanup in the destructor. But there are key differences:

```
                unique_ptr + custom deleter         ScopeGuard
                ───────────────────────────         ──────────
Resource type   Must be a pointer (T*)              Any cleanup — no pointer needed
Dismiss?        No built-in dismiss                 Yes — dismiss() skips cleanup
Access?         Provides get(), *, ->               No resource access — just cleanup
Use case        Managing a heap object or handle    Arbitrary rollback / cleanup logic
```

Use `unique_ptr` when you have a **pointer-like resource** you need to access through the wrapper. Use `ScopeGuard` when you just need to **run some code** on scope exit (rollback, logging, unlocking).

<br>

#### Can you fake a ScopeGuard with unique_ptr?

```cpp
auto guard = std::unique_ptr<void, std::function<void(void*)>>(
    (void*)1,   // dummy non-null pointer
    [](void*) { cleanup(); }
);
```

This works but is ugly and allocates (for `std::function`). `ScopeGuard` is cleaner and zero-overhead.

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 Q2: Why store the callable by value, not by reference?</h2>

If you stored a reference to the lambda:

```cpp
template <typename F>
class ScopeGuard {
    F& func_;   // DANGER — reference
};
```

The lambda might be a temporary (rvalue). Temporaries die at the end of the statement. Your reference would dangle:

```cpp
auto guard = ScopeGuard([] { cleanup(); });
//           ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
//           lambda is a temporary — dies after this line
//           guard.func_ is now a dangling reference!
```

By storing **by value**, the lambda is moved/copied into the guard and lives as long as the guard does:

```cpp
template <typename F>
class ScopeGuard {
    F func_;   // VALUE — owns the callable, no dangling
};
```

This is the same reason `std::function`, `std::thread`, and `std::packaged_task` all store callables by value.

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 Q3: Why does the move constructor need to deactivate the source?</h2>

If you move a ScopeGuard without deactivating the source:

```cpp
ScopeGuard(ScopeGuard&& other) noexcept
    : func_(std::move(other.func_))
    , active_(other.active_)
{
    // forgot: other.active_ = false;
}
```

Now both `this` and `other` have `active_ = true`. When both are destroyed, the cleanup runs **twice** — double rollback, double close, double free.

```cpp
auto a = make_scope_guard([] { close(fd); });
auto b = std::move(a);
// a is destroyed → close(fd)
// b is destroyed → close(fd)   ← DOUBLE CLOSE!
```

Always deactivate the source on move:

```cpp
ScopeGuard(ScopeGuard&& other) noexcept
    : func_(std::move(other.func_))
    , active_(other.active_)
{
    other.active_ = false;   // source won't run cleanup
}
```

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 Q4: FileHandle vs <code>unique_ptr&lt;int, close_deleter&gt;</code> — why not just use unique_ptr?</h2>

You can't use `unique_ptr` directly because file descriptors are **integers**, not pointers. `unique_ptr` requires a pointer type (`T*`):

```cpp
// This doesn't work:
std::unique_ptr<int, ???> fd;   // int is not a pointer type
```

You could hack it with `intptr_t` casts, but it's ugly and error-prone. A dedicated `FileHandle` class is cleaner because:

1. The "empty" state is `-1`, not `nullptr` (0 is a valid fd — stdin!)
2. You can add convenience methods (`read`, `write`, `seek`)
3. The interface communicates intent — `FileHandle` is self-documenting

This is why the C++ standards committee proposed `std::experimental::unique_resource` — a generalized RAII wrapper for **any** resource type, not just pointers.

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 Q5: What happens if the destructor cleanup throws an exception?</h2>

Short answer: **`std::terminate` is called and your program crashes.**

In C++11+, destructors are implicitly `noexcept`. If an exception escapes a `noexcept` function, `std::terminate()` is invoked — no unwinding, no catch, just immediate death.

Even if you explicitly mark the destructor as `noexcept(false)`, throwing from a destructor is extremely dangerous:

```
Scenario: An exception is thrown.
  → Stack unwinding begins.
  → Destructors run for local objects.
  → One destructor throws ANOTHER exception.
  → Two exceptions in flight simultaneously.
  → C++ cannot handle this → std::terminate().
```

**Rule**: Never throw from a destructor. If your cleanup can fail (e.g., `close()` can return an error), either:
- Ignore the error (common for file descriptors)
- Log it and swallow the exception
- Provide an explicit `close()` method the user can call before destruction to handle errors

```cpp
~FileHandle() {
    if (fd_ >= 0) {
        int ret = ::close(fd_);
        if (ret == -1) {
            // Log but DON'T throw
            std::cerr << "close() failed: " << strerror(errno) << "\n";
        }
    }
}
```

<br><br>

---
---

# PART 5: REFLECT & NOTES (10 min)

---

<br>

1. **What is RAII in one sentence?**
   - Acquire resources in constructors, release in destructors — automatic cleanup on scope exit.

2. **When do you use ScopeGuard vs unique_ptr with custom deleter?**
   - ScopeGuard for arbitrary cleanup logic. `unique_ptr` for pointer-like resources you need to access.

3. **Why is ScopeGuard move-only?**
   - Copying would run cleanup twice. Only one owner should be responsible for the cleanup.

4. **Why can't file descriptors use `unique_ptr` directly?**
   - FDs are integers, not pointers. `0` is valid (stdin), so `nullptr` can't represent "empty". Need `-1` as sentinel.

5. **What happens if a destructor throws?**
   - `std::terminate` is called. Never throw from destructors.

<br><br>

---
---

# INTERVIEW QUESTIONS TO PRACTICE

---

<br>

1. "What is RAII? Give three examples from the standard library."
2. "Implement a ScopeGuard that runs cleanup on scope exit."
3. "Why should destructors never throw? What happens if they do?"
4. "Design an RAII wrapper for a POSIX file descriptor."
5. "How would you implement transactional rollback using RAII?"
6. "What is the difference between `unique_ptr` with a custom deleter and a ScopeGuard?"
7. "What is `std::lock_guard`? How is it an example of RAII?"

<br><br>

---
---

# REFERENCE SOLUTION

---

<br>

<details>
<summary>Click to expand the complete ScopeGuard + FileHandle implementation</summary>

<br>

```cpp
#include <iostream>
#include <utility>
#include <fcntl.h>
#include <unistd.h>
#include <cerrno>
#include <cstring>
#include <stdexcept>
#include <cassert>

// ========================
// ScopeGuard
// ========================

template <typename F>
class ScopeGuard {
    F func_;
    bool active_;

public:
    explicit ScopeGuard(F f) noexcept
        : func_(std::move(f)), active_(true) {}

    ~ScopeGuard() {
        if (active_) func_();
    }

    ScopeGuard(const ScopeGuard&) = delete;
    ScopeGuard& operator=(const ScopeGuard&) = delete;

    ScopeGuard(ScopeGuard&& other) noexcept
        : func_(std::move(other.func_)), active_(other.active_)
    {
        other.active_ = false;
    }

    ScopeGuard& operator=(ScopeGuard&&) = delete;

    void dismiss() noexcept { active_ = false; }
};

template <typename F>
ScopeGuard<F> make_scope_guard(F&& f) {
    return ScopeGuard<std::decay_t<F>>(std::forward<F>(f));
}

// ========================
// FileHandle
// ========================

class FileHandle {
    int fd_;

public:
    explicit FileHandle(int fd = -1) noexcept : fd_(fd) {}

    FileHandle(const char* path, int flags, mode_t mode = 0644)
        : fd_(::open(path, flags, mode))
    {
        if (fd_ < 0) {
            std::cerr << "  open failed: " << strerror(errno) << "\n";
        }
    }

    ~FileHandle() {
        if (fd_ >= 0) {
            std::cout << "  [~FileHandle] closing fd " << fd_ << "\n";
            ::close(fd_);
        }
    }

    FileHandle(const FileHandle&) = delete;
    FileHandle& operator=(const FileHandle&) = delete;

    FileHandle(FileHandle&& other) noexcept : fd_(other.fd_) {
        other.fd_ = -1;
    }

    FileHandle& operator=(FileHandle&& other) noexcept {
        if (this != &other) {
            reset(other.fd_);
            other.fd_ = -1;
        }
        return *this;
    }

    int fd() const noexcept { return fd_; }
    bool is_valid() const noexcept { return fd_ >= 0; }
    explicit operator bool() const noexcept { return is_valid(); }

    int release() noexcept {
        int tmp = fd_;
        fd_ = -1;
        return tmp;
    }

    void reset(int new_fd = -1) noexcept {
        int old = fd_;
        fd_ = new_fd;
        if (old >= 0) ::close(old);
    }

    friend void swap(FileHandle& a, FileHandle& b) noexcept {
        using std::swap;
        swap(a.fd_, b.fd_);
    }
};

// ========================
// Tests
// ========================

int main() {
    std::cout << "=== ScopeGuard: basic ===\n";
    {
        auto g = make_scope_guard([] { std::cout << "  Cleanup!\n"; });
        std::cout << "  Inside scope\n";
    }

    std::cout << "\n=== ScopeGuard: dismiss ===\n";
    {
        auto g = make_scope_guard([] { std::cout << "  SHOULD NOT PRINT\n"; });
        g.dismiss();
    }
    std::cout << "  Dismissed OK\n";

    std::cout << "\n=== ScopeGuard: exception ===\n";
    try {
        auto g = make_scope_guard([] { std::cout << "  Cleanup on throw!\n"; });
        throw std::runtime_error("test");
    } catch (...) {}

    std::cout << "\n=== ScopeGuard: transaction ===\n";
    {
        int balance = 100;
        auto withdraw = [&](int amt) {
            balance -= amt;
            auto rollback = make_scope_guard([&] { balance += amt; });
            if (amt > 50) { std::cout << "  Failed, rolling back\n"; return; }
            rollback.dismiss();
            std::cout << "  Succeeded\n";
        };
        withdraw(30);
        withdraw(80);
        std::cout << "  Balance: " << balance << "\n";
        assert(balance == 70);
    }

    std::cout << "\n=== FileHandle ===\n";
    {
        FileHandle f(::open("/dev/null", O_RDONLY));
        std::cout << "  fd = " << f.fd() << "\n";
        FileHandle g = std::move(f);
        std::cout << "  after move: f=" << f.fd() << " g=" << g.fd() << "\n";
    }

    std::cout << "\n=== Done ===\n";
    return 0;
}
```

</details>

<br>

---

**Next**: [Day 6 — Copy-on-Write & Pimpl →](day06-cow-pimpl.md)  *(coming soon)*
