# OS Appendix 3: Context Switching & Scheduling

[← Back to OS Appendix](README.md) | [← Process Model](02-process-model.md)

---
---

# CONTEXT SWITCHING, SYSCALLS & CPU SCHEDULING

---

<br>

<h2 style="color: #2980B9;">📘 3.1 User Mode vs Kernel Mode</h2>

The CPU has (at minimum) two privilege levels:

```
┌─────────────────────────────────────────────────────────┐
│ User Mode (Ring 3)                                      │
│                                                         │
│  Your C++ code runs here.                               │
│  ✗ Cannot access hardware directly                      │
│  ✗ Cannot access other processes' memory                │
│  ✗ Cannot execute privileged instructions (e.g., halt)  │
│  ✓ Can access your own memory                           │
│  ✓ Can do computation (pure math, logic)                │
└────────────────────────┬────────────────────────────────┘
                         │ system call (trap)
                         ▼
┌─────────────────────────────────────────────────────────┐
│ Kernel Mode (Ring 0)                                    │
│                                                         │
│  OS kernel runs here.                                   │
│  ✓ Full access to hardware                              │
│  ✓ Can access ALL physical memory                       │
│  ✓ Can modify page tables, interrupt handlers           │
│  ✓ Runs device drivers                                  │
└─────────────────────────────────────────────────────────┘
```

Transitioning between modes (via **system calls**) is expensive — typically 100-1000 cycles.

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 3.2 System Calls — How User Code Talks to the Kernel</h2>

When your C++ code calls `read()`, `write()`, `open()`, `close()`, `fork()`, `mmap()`, etc., it's making a **system call** — a request for the kernel to do something privileged:

```
Your code:                              What actually happens:
─────────                               ──────────────────────
read(fd, buf, 100);                     1. libc puts syscall # in register
                                        2. Executes SYSCALL instruction
                                        3. CPU switches to kernel mode
                                        4. Kernel validates arguments
                                        5. Kernel performs the I/O
                                        6. Kernel copies data to user buffer
                                        7. CPU switches back to user mode
                                        8. read() returns
```

**Every syscall has overhead**: save/restore registers, switch privilege level, validate arguments, switch back. This is why:

- `read()` with 1 byte at a time is slow → batch reads with `fread()` or large buffers
- Lock/unlock a mutex → `futex` syscall (only in contended case — fast path is user-space atomic)
- Allocating memory → `mmap`/`brk` syscall (malloc batches these)

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 3.3 What Is a Context Switch?</h2>

When the OS switches the CPU from running one process/thread to another:

```
Process A running                      Process B running
┌────────────────┐                     ┌────────────────┐
│ Registers:     │                     │ Registers:     │
│  PC = 0x401234 │                     │  PC = 0x502000 │
│  SP = 0x7FFE00 │                     │  SP = 0x7FFC00 │
│  RAX = 42      │                     │  RAX = 7       │
│  ...           │                     │  ...           │
└────────────────┘                     └────────────────┘

Context Switch A → B:
  1. Save A's registers to A's kernel stack / task struct
  2. Save A's page table pointer (CR3 on x86)
  3. Load B's registers from B's saved state
  4. Load B's page table pointer (CR3)
  5. Flush TLB (for process switch — not needed for thread switch within same process)
  6. Resume B from where it left off
```

**Cost of a context switch:**

| Operation | Cost |
|-----------|------|
| Save/restore registers | ~100 cycles |
| Switch page table (CR3) | ~100 cycles |
| TLB flush + refill | ~1000-10000 cycles (depends on working set) |
| Cache pollution | Hard to measure — can be millions of cycles in worst case |

**Thread switch within same process** is cheaper — no page table swap, no TLB flush (threads share address space).

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 3.4 When Do Context Switches Happen?</h2>

| Trigger | Type | Example |
|---------|------|---------|
| **Timer interrupt** | Preemptive | Time slice expired (e.g., 4ms on Linux CFS) |
| **System call** | Voluntary | `read()` blocks waiting for I/O |
| **I/O completion** | Interrupt | Disk read finished → wake up blocked process |
| **`sched_yield()`** | Voluntary | Thread explicitly gives up CPU |
| **Higher priority ready** | Preemptive | Real-time task becomes runnable |
| **Mutex/cond_var wait** | Voluntary | Thread blocks on `pthread_mutex_lock()` |

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 3.5 Linux CFS Scheduler</h2>

Linux uses the **Completely Fair Scheduler (CFS)** for normal processes:

- Each task has a **virtual runtime** (`vruntime`) — how much CPU time it has used
- CFS always picks the task with the **smallest vruntime** (the one that has run the least)
- Implemented with a **red-black tree** keyed by vruntime — O(log n) to find next task
- Default time slice: ~4ms (varies with number of runnable tasks)

```
Red-black tree of runnable tasks (sorted by vruntime):

        vruntime=50
       /           \
  vruntime=30    vruntime=70
    /     \
vruntime=20  vruntime=40

Next to run: leftmost node = vruntime=20 (has run the least)
```

**Nice values** (-20 to +19) adjust the rate at which vruntime increases:
- Nice -20 (highest priority): vruntime grows slowly → gets more CPU
- Nice +19 (lowest priority): vruntime grows quickly → gets less CPU

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 3.6 Why Context Switching Matters for C++ Engineers</h2>

Understanding context switch cost explains many performance decisions:

| Decision | Reason |
|----------|--------|
| Use thread pool instead of thread-per-request | Fewer threads → fewer context switches |
| Use `epoll`/`kqueue` instead of blocking I/O | One thread handles thousands of connections, no switching |
| Use lock-free data structures | Avoid mutex → avoid syscall → avoid context switch |
| Use `spinlock` for very short critical sections | Spinning wastes CPU but avoids context switch overhead |
| Batch small I/O operations | Fewer syscalls → fewer user/kernel transitions |
| Use `mmap` instead of `read`/`write` | Avoids syscall per access (page faults are cheaper in bulk) |

<br><br>

---

## Interview Questions

1. "What is a context switch? What gets saved and restored?"
2. "Why is a thread context switch cheaper than a process context switch?"
3. "What is the difference between user mode and kernel mode?"
4. "What is a system call? Why is it expensive?"
5. "How does the Linux CFS scheduler work?"
6. "Why do we use thread pools instead of creating threads per request?"
7. "When is a spinlock better than a mutex?"

---

**Next**: [Signals →](04-signals.md)
