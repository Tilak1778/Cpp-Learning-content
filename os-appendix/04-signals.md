# OS Appendix 4: Signals

[← Back to OS Appendix](README.md) | [← Context Switching](03-context-switching.md)

---
---

# SIGNAL HANDLING IN LINUX

---

<br>

<h2 style="color: #2980B9;">📘 4.1 What Is a Signal?</h2>

A signal is a **software interrupt** delivered to a process. It's the OS's way of telling a process "something happened" — asynchronously, at any point during execution.

```
Kernel or another process
        │
    sends signal
        │
        ▼
┌──────────────────────────────────┐
│         Target Process           │
│                                  │
│  Currently executing some code   │
│            │                     │
│     ◄──── INTERRUPTED ────►      │
│            │                     │
│     Signal handler runs          │
│     (or default action taken)    │
│            │                     │
│     ◄──── RESUMED ─────►        │
│                                  │
└──────────────────────────────────┘
```

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 4.2 Common Signals</h2>

| Signal | Number | Default Action | When Sent |
|--------|--------|----------------|-----------|
| `SIGTERM` | 15 | Terminate | Polite "please exit" (`kill <pid>`) |
| `SIGKILL` | 9 | Terminate | Forced kill — **cannot be caught or ignored** |
| `SIGINT` | 2 | Terminate | Ctrl+C in terminal |
| `SIGSEGV` | 11 | Core dump | Invalid memory access (null ptr, bad ptr) |
| `SIGBUS` | 7/10 | Core dump | Bus error (misaligned access, bad mmap) |
| `SIGFPE` | 8 | Core dump | Arithmetic error (division by zero) |
| `SIGABRT` | 6 | Core dump | `abort()` called (assert failure) |
| `SIGPIPE` | 13 | Terminate | Write to a closed pipe/socket |
| `SIGCHLD` | 17/20 | Ignore | Child process exited |
| `SIGSTOP` | 19/23 | Stop | Pause process — **cannot be caught** |
| `SIGCONT` | 18/25 | Continue | Resume paused process |
| `SIGUSR1` | 10/30 | Terminate | User-defined (custom use) |
| `SIGUSR2` | 12/31 | Terminate | User-defined (custom use) |
| `SIGALRM` | 14 | Terminate | Timer expired (`alarm()`) |
| `SIGHUP` | 1 | Terminate | Terminal closed / daemon reload convention |

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 4.3 Installing a Signal Handler</h2>

#### The modern way: `sigaction()`

```cpp
#include <signal.h>
#include <unistd.h>
#include <cstring>
#include <cstdio>

volatile sig_atomic_t g_shutdown = 0;

void handle_sigterm(int signum) {
    // This runs ASYNCHRONOUSLY — can interrupt ANY instruction
    g_shutdown = 1;
    // Only async-signal-safe functions allowed here!
}

int main() {
    struct sigaction sa;
    std::memset(&sa, 0, sizeof(sa));
    sa.sa_handler = handle_sigterm;
    sigemptyset(&sa.sa_mask);
    sa.sa_flags = 0;

    sigaction(SIGTERM, &sa, nullptr);
    sigaction(SIGINT, &sa, nullptr);

    printf("PID: %d — send me SIGTERM or press Ctrl+C\n", getpid());

    while (!g_shutdown) {
        // Main work loop
        sleep(1);
        printf("Working...\n");
    }

    printf("Shutting down gracefully.\n");
    return 0;
}
```

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 4.4 The Async-Signal-Safety Problem</h2>

**This is the single most important thing to understand about signals.**

A signal handler can interrupt your program at **any** instruction — even in the middle of `malloc`, `printf`, or updating a data structure. If the handler calls the same function, you get:

- Deadlock (if the function holds a lock)
- Corruption (if the function is modifying shared state)

```
Thread is inside malloc():
  malloc() holds an internal lock
  malloc() is updating the free list
       │
  ◄── SIGNAL arrives, handler runs ──►
       │
  Handler calls printf() → calls malloc() → DEADLOCK!
  (malloc tries to acquire the same lock it already holds)
```

Only **async-signal-safe** functions can be called inside a signal handler. The POSIX-guaranteed safe list includes:

| Safe | NOT Safe |
|------|----------|
| `write()` | `printf()`, `fprintf()` |
| `_exit()` | `exit()` (runs atexit handlers) |
| `signal()`, `sigaction()` | `malloc()`, `free()`, `new`, `delete` |
| `read()` | `std::cout`, `std::string` |
| `open()`, `close()` | Any C++ STL function |
| `getpid()` | `pthread_mutex_lock()` |

**Rule of thumb**: In a signal handler, only set a flag (`volatile sig_atomic_t`) and return. Do the real work in the main loop.

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 4.5 <code>volatile sig_atomic_t</code> — Why Both Keywords?</h2>

```cpp
volatile sig_atomic_t g_flag = 0;
```

- **`volatile`** — tells the compiler "don't optimize away reads of this variable; it can change at any time" (because the signal handler modifies it asynchronously)
- **`sig_atomic_t`** — an integer type guaranteed to be read/written atomically with respect to signals (typically just an `int`). This is NOT `std::atomic` — it only guarantees atomicity against **signal interruption within the same thread**, not across threads.

```
Without volatile:
  while (!g_flag) { ... }
  Compiler may optimize to:
    if (!g_flag) { while (true) { ... } }   // never re-reads g_flag!

With volatile:
  Compiler must re-read g_flag from memory every iteration.
```

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 4.6 <code>signalfd()</code> — The Modern Alternative (Linux)</h2>

Instead of dealing with async signal handlers, Linux offers `signalfd()` — it converts signals into **file descriptor events** that you can read synchronously:

```cpp
#include <sys/signalfd.h>
#include <signal.h>
#include <unistd.h>

int main() {
    sigset_t mask;
    sigemptyset(&mask);
    sigaddset(&mask, SIGTERM);
    sigaddset(&mask, SIGINT);

    // Block these signals from being delivered normally
    sigprocmask(SIG_BLOCK, &mask, nullptr);

    // Create a file descriptor that receives these signals
    int sfd = signalfd(-1, &mask, 0);

    // Now you can read signals like reading from a file:
    struct signalfd_siginfo info;
    read(sfd, &info, sizeof(info));  // blocks until signal arrives

    printf("Got signal %d\n", info.ssi_signo);
    // Handle it here — synchronously, safely, no restrictions!

    close(sfd);
    return 0;
}
```

Benefits:
- No async-signal-safety worries — you handle signals in your main event loop
- Works with `epoll` — multiplex signals with socket/file I/O
- Used by modern servers (e.g., systemd, nginx)

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 4.7 Signals and Multithreading</h2>

Signals in multithreaded programs are tricky:

```
Process with 3 threads:
┌──────────────────────────────────┐
│ Thread 1   Thread 2   Thread 3  │
│                                  │
│  SIGTERM arrives — which thread  │
│  handles it??                    │
└──────────────────────────────────┘

Rules:
  - Process-directed signals (kill -TERM <pid>): delivered to ANY thread
    that doesn't have the signal blocked
  - Thread-directed signals (pthread_kill): delivered to specific thread
  - SIGSEGV/SIGFPE/SIGBUS: delivered to the thread that caused it
```

**Best practice for multithreaded servers:**
1. Block all signals in all threads at startup
2. Create one dedicated signal-handling thread
3. That thread uses `sigwait()` or `signalfd()` to receive signals synchronously

```cpp
void signal_thread_func(sigset_t mask) {
    int sig;
    while (true) {
        sigwait(&mask, &sig);
        if (sig == SIGTERM) {
            // Set a shared atomic flag
            g_shutdown.store(true, std::memory_order_release);
            break;
        }
    }
}
```

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 4.8 SIGPIPE — The Silent Server Killer</h2>

When you write to a socket/pipe whose other end is closed, the kernel sends `SIGPIPE`. Default action: **terminate the process**.

This is a common surprise in server code — one disconnected client kills the entire server.

**Fix**: Ignore `SIGPIPE` at startup:

```cpp
signal(SIGPIPE, SIG_IGN);
// Now write() returns -1 with errno=EPIPE instead of killing the process
```

Or use `MSG_NOSIGNAL` flag with `send()`:
```cpp
send(fd, buf, len, MSG_NOSIGNAL);
```

<br><br>

---

## Interview Questions

1. "What is a signal? How is it different from an exception?"
2. "What functions are safe to call in a signal handler? Why?"
3. "How would you implement graceful shutdown in a C++ server?"
4. "What is the difference between `SIGTERM` and `SIGKILL`?"
5. "How do you handle signals in a multithreaded program?"
6. "What is `SIGPIPE`? Why is it dangerous in server code?"
7. "What is `signalfd()`? Why is it better than traditional signal handlers?"
8. "Why do we need both `volatile` and `sig_atomic_t`?"

---

**Next**: [Memory-Mapped I/O →](05-mmap.md)
