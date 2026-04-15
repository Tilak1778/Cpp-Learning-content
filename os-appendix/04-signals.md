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

## Minix 3 Deep Dive — Signal Delivery in a Microkernel

Signal handling in Minix 3 perfectly illustrates the microkernel philosophy: the **kernel** only handles the bare minimum (notifications), while the **PM server** manages all signal policy (which signals, which handler, masking).

<br>

<h3 style="color: #E67E22;">Signal Handling Source Files</h3>

```
minix/servers/pm/
├── signal.c      ← the main signal logic: delivery, checking, sigaction
├── mproc.h       ← per-process signal state (pending, mask, handlers)
└── forkexit.c    ← exit on fatal signals, signal during wait()

minix/kernel/
├── system/
│   ├── do_sigsend.c   ← kernel pushes signal frame onto user stack
│   ├── do_sigreturn.c ← kernel restores state after handler returns
│   └── do_kill.c      ← kernel-level signal notification
└── proc.c             ← SIGPENDING flag in process state
```

<br>

<h3 style="color: #E67E22;">How Signal Delivery Works — The Full Path</h3>

In Linux, the kernel directly manipulates the process's stack to invoke the signal handler. In Minix 3, PM orchestrates the whole thing:

```
Step 1: Signal originates
  - User calls kill(pid, SIGTERM)
  - Or hardware exception (SIGSEGV)
  - Or kernel event (SIGCHLD on child exit)
        │
        ▼
Step 2: PM receives the signal request
  PM looks up the target process's signal state:
  - Is this signal blocked (in mp_sigmask)? → add to mp_sigpending, done
  - Is this signal ignored (SIG_IGN)? → discard, done
  - Is this signal caught (handler installed)? → go to step 3
  - Default action? → terminate, core dump, stop, or ignore
        │
        ▼
Step 3: PM asks kernel to set up the signal frame
  PM calls sys_sigsend() → kernel call SYS_SIGSEND
  Kernel modifies the target process's saved stack:
  - Pushes a sigcontext (saved registers) onto user stack
  - Sets process's PC to the signal handler address
  - Sets process's SP to the new stack frame
  - When the process runs, it executes the handler
        │
        ▼
Step 4: Signal handler runs (in user space)
  Handler function executes with the pushed sigcontext below it
        │
        ▼
Step 5: sigreturn()
  When the handler returns, libc calls sigreturn()
  → kernel call SYS_SIGRETURN
  Kernel restores the original registers from the sigcontext
  Process resumes where it was interrupted
```

<br>

<h3 style="color: #E67E22;">PM's Signal Processing Code</h3>

```c
/* servers/pm/signal.c (simplified) */

int do_kill(void) {
    pid_t pid = m_in.m_lc_pm_sig.pid;
    int signo = m_in.m_lc_pm_sig.sig;

    /* Find the target process */
    struct mproc *target = find_proc(pid);

    /* Permission check: can the caller send this signal? */
    if (!check_sig_permission(mp, target, signo))
        return EPERM;

    /* Process the signal */
    sig_proc(target, signo, TRUE);

    return OK;
}

void sig_proc(struct mproc *rmp, int signo, int trace) {
    /* Is the signal blocked? */
    if (sigismember(&rmp->mp_sigmask, signo)) {
        /* Add to pending set — will be delivered when unblocked */
        sigaddset(&rmp->mp_sigpending, signo);
        return;
    }

    /* Check what action is configured */
    struct sigaction *sa = &rmp->mp_sigact[signo];

    if (sa->sa_handler == SIG_IGN) {
        return;  /* ignored */
    }

    if (sa->sa_handler == SIG_DFL) {
        /* Default action */
        switch (signo) {
            case SIGTERM: case SIGKILL: case SIGINT:
                /* Terminate the process */
                pm_exit(rmp, signo);
                return;
            case SIGSTOP: case SIGTSTP:
                /* Stop the process */
                rmp->mp_flags |= STOPPED;
                sys_stop(rmp->mp_endpoint);
                return;
            case SIGCHLD: case SIGURG:
                return;  /* default is ignore */
            /* ... */
        }
    }

    /* Custom handler — ask the kernel to set up the stack frame */
    struct sigmsg smsg;
    smsg.sm_signo = signo;
    smsg.sm_sighandler = (vir_bytes)sa->sa_handler;
    smsg.sm_mask = sa->sa_mask;  /* mask during handler execution */
    smsg.sm_sigreturn = (vir_bytes)__sigreturn_stub;

    sys_sigsend(rmp->mp_endpoint, &smsg);

    /* Block this signal during handler execution (unless SA_NODEFER) */
    if (!(sa->sa_flags & SA_NODEFER)) {
        sigaddset(&rmp->mp_sigmask, signo);
    }
}
```

<br>

<h3 style="color: #E67E22;">Kernel-Side Signal Frame Setup</h3>

The kernel's `do_sigsend()` is where the magic happens — it modifies the process's saved registers to redirect execution to the signal handler:

```c
/* kernel/system/do_sigsend.c (simplified) */
int do_sigsend(struct proc *caller, message *msg) {
    struct proc *target = proc_addr(msg->SIG_ENDPT);
    struct sigmsg *smsg = &msg->SIG_MSG;

    /* Save current register state to a sigcontext on the user stack */
    struct sigcontext sc;
    sc.sc_eip = target->p_reg.pc;       /* where the process was executing */
    sc.sc_esp = target->p_reg.sp;       /* original stack pointer */
    sc.sc_eax = target->p_reg.retreg;   /* return value register */
    sc.sc_mask = target->p_sigmask;     /* original signal mask */
    /* ... save all other registers ... */

    /* Push sigcontext onto the process's user stack */
    vir_bytes new_sp = target->p_reg.sp - sizeof(struct sigcontext);
    data_copy(KERNEL, &sc, target->p_endpoint, new_sp, sizeof(sc));

    /* Push arguments for the signal handler: (signo, &sigcontext) */
    new_sp -= sizeof(int);
    data_copy(KERNEL, &smsg->sm_signo, target->p_endpoint, new_sp, sizeof(int));

    /* Push return address (points to sigreturn stub) */
    new_sp -= sizeof(vir_bytes);
    data_copy(KERNEL, &smsg->sm_sigreturn, target->p_endpoint, new_sp,
              sizeof(vir_bytes));

    /* Redirect execution to the signal handler */
    target->p_reg.pc = smsg->sm_sighandler;  /* jump to handler */
    target->p_reg.sp = new_sp;               /* use new stack */

    return OK;
}
```

When the handler returns, the `sigreturn` stub calls `SYS_SIGRETURN`, and the kernel's `do_sigreturn()` restores the original registers from the sigcontext on the stack — seamlessly resuming the interrupted code.

<br>

<h3 style="color: #E67E22;">SIGKILL and SIGSTOP — Uncatchable Signals</h3>

Even in Minix 3's PM, `SIGKILL` and `SIGSTOP` bypass the handler logic entirely:

```c
/* servers/pm/signal.c */
if (signo == SIGKILL) {
    /* Cannot be caught, blocked, or ignored — terminate immediately */
    pm_exit(rmp, SIGKILL);
    return;
}
if (signo == SIGSTOP) {
    /* Cannot be caught — stop immediately */
    sys_stop(rmp->mp_endpoint);
    rmp->mp_flags |= STOPPED;
    return;
}
```

<br>

<h3 style="color: #E67E22;">Hardware Exceptions → Signals</h3>

When the CPU raises an exception (like a null pointer dereference), the kernel translates it into a signal notification to PM:

```
CPU exception (e.g., page fault at address 0x0):
  kernel/arch/i386/exception.c:
  ↓
  cause_sig(proc_nr, SIGSEGV)     /* notify PM about this signal */
  ↓
  PM receives notification
  ↓
  sig_proc(target, SIGSEGV, ...)
  ↓
  Default action: terminate + core dump
```

```c
/* kernel/proc.c (simplified) */
void cause_sig(proc_nr_t proc_nr, int signo) {
    struct proc *rp = proc_addr(proc_nr);

    /* Mark process as having a pending signal */
    rp->p_rts_flags |= SIG_PENDING;

    /* Notify PM server that this process has a signal */
    send_sig(PM_PROC_NR, SIGKSIGSM);
}
```

<br>

<h3 style="color: #E67E22;">Microkernel vs Monolithic — Signal Comparison</h3>

| Aspect | Linux (Monolithic) | Minix 3 (Microkernel) |
|--------|-------------------|----------------------|
| Signal state | In `task_struct` (kernel) | In `mproc` (PM server, user-space) |
| Handler dispatch | Kernel directly modifies user stack | PM instructs kernel via SYS_SIGSEND |
| Permission checks | Kernel checks (single function) | PM checks (separate server) |
| sigaction() | Kernel syscall handler | PM message handler |
| Performance | Fast (no IPC) | Slower (PM ↔ kernel messages) |
| Crash safety | Bug in signal code → kernel panic | Bug in PM signal code → PM restart |
| Code clarity | Mixed with scheduler & memory | Cleanly separated in `signal.c` |

The Minix 3 approach makes the signal delivery mechanism extremely transparent — you can literally trace each message and see exactly what happens. In Linux, the equivalent code in `kernel/signal.c` is interleaved with scheduler, thread group, and namespace logic.

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
9. "Walk through how a signal handler is invoked at the stack frame level."
10. "How does Minix 3 deliver signals differently from Linux?"

---

**Next**: [Memory-Mapped I/O →](05-mmap.md)
