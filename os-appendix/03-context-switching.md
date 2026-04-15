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

## Minix 3 Deep Dive — Context Switching & System Calls

Minix 3's approach to context switching and system calls is fundamentally different from Linux, and studying it reveals what "monolithic vs microkernel" really means at the code level.

<br>

<h3 style="color: #E67E22;">System Calls in Minix 3 — Message Passing</h3>

In Linux, a system call traps directly into the kernel, which executes the handler and returns. In Minix 3, a system call is a **message** sent to a server:

```
Linux system call:
  User code → SYSCALL instruction → kernel handler → result → return to user
  (one context switch: user → kernel → user)

Minix 3 system call:
  User code → SYSCALL instruction → kernel receives message
  → kernel delivers message to PM/VFS/VM server
  → server processes it, sends reply message
  → kernel delivers reply to user process
  (multiple context switches: user → kernel → server → kernel → user)
```

The key file is `minix/kernel/proc.c`, which implements the IPC primitives:

```c
/* kernel/proc.c — the heart of Minix 3's microkernel (simplified) */

int mini_send(struct proc *caller, int dst, message *msg, int flags) {
    struct proc *dst_proc = proc_addr(dst);

    /* Is the destination waiting for a message from us? */
    if ((dst_proc->p_rts_flags & RECEIVING) &&
        (dst_proc->p_getfrom == caller->p_nr || dst_proc->p_getfrom == ANY)) {
        /* Destination is blocked in receive() — deliver directly */
        copy_msg(caller, msg, dst_proc, dst_proc->p_messbuf);

        /* Unblock the destination */
        dst_proc->p_rts_flags &= ~RECEIVING;
        enqueue(dst_proc);  /* put it back on ready queue */
    } else {
        /* Destination is NOT waiting — block the sender */
        caller->p_rts_flags |= SENDING;
        caller->p_sendto = dst;

        /* Add sender to destination's send queue */
        enqueue_sender(dst_proc, caller);
    }

    return OK;
}

int mini_receive(struct proc *caller, int src, message *msg, int flags) {
    /* Check if anyone is already waiting to send to us */
    struct proc *sender = dequeue_sender(caller, src);

    if (sender) {
        /* Found a waiting sender — grab their message */
        copy_msg(sender, sender->p_messbuf, caller, msg);

        /* Unblock the sender */
        sender->p_rts_flags &= ~SENDING;
        enqueue(sender);
    } else {
        /* Nobody waiting — block until a message arrives */
        caller->p_rts_flags |= RECEIVING;
        caller->p_getfrom = src;
        caller->p_messbuf = msg;
    }

    return OK;
}
```

This is the **rendezvous** IPC model — the sender blocks until the receiver is ready (synchronous message passing). Every system call in Minix 3 ultimately goes through `mini_send` and `mini_receive`.

<br>

<h3 style="color: #E67E22;">The Message Structure</h3>

All communication between processes, servers, and the kernel uses a fixed-size message:

```c
/* include/minix/ipc.h (simplified) */
typedef struct {
    int m_source;     /* who sent this message */
    int m_type;       /* message type (syscall number, reply, notification) */
    union {
        mess_1 m_m1;  /* variant 1: 3 ints + 3 pointers */
        mess_2 m_m2;  /* variant 2: 3 ints + 2 pointers + array */
        mess_3 m_m3;  /* variant 3: ... */
        /* ... several variants for different syscalls */
    } m_u;
} message;
```

For example, a `read()` system call becomes:

```
User calls: read(fd, buf, 100)
  ↓
libc packs a message:
  msg.m_type  = VFS_READ
  msg.m_m1.fd = fd
  msg.m_m1.buf = buf
  msg.m_m1.count = 100
  ↓
sendrec(VFS_ENDPOINT, &msg)   ← send to VFS server and wait for reply
  ↓
VFS server receives the message, does the I/O, sends reply:
  reply_msg.m_type = OK
  reply_msg.m_m1.count = bytes_read
  ↓
User process unblocks, read() returns bytes_read
```

<br>

<h3 style="color: #E67E22;">Minix 3 Source Tree — Kernel & Scheduling</h3>

```
minix/kernel/
├── proc.c          ← IPC (mini_send, mini_receive), scheduling (enqueue/dequeue)
├── proc.h          ← process table entry, scheduling state
├── clock.c         ← timer interrupt handler, time quantum management
├── interrupt.c     ← hardware interrupt handling
├── system.c        ← kernel calls dispatch (SYS_FORK, SYS_EXEC, etc.)
├── system/
│   ├── do_fork.c   ← kernel-side fork
│   ├── do_exec.c   ← kernel-side exec (reset registers)
│   ├── do_copy.c   ← inter-process memory copy
│   └── do_irqctl.c ← interrupt control for user-space drivers
├── arch/i386/
│   ├── mpx.S       ← low-level context switch (save/restore registers)
│   ├── exception.c ← CPU exception handlers
│   └── memory.c    ← page table manipulation
└── table.c         ← system call dispatch table
```

<br>

<h3 style="color: #E67E22;">Context Switch in Minix 3</h3>

The actual register save/restore happens in the assembly file `kernel/arch/i386/mpx.S`:

```asm
; kernel/arch/i386/mpx.S (simplified)

; Save current process's context
save_context:
    push eax
    push ebx
    push ecx
    push edx
    push esi
    push edi
    push ebp

    ; Save stack pointer to current process's proc structure
    mov [current_proc + P_STACKFRAME + SP_OFF], esp

    ; Load kernel stack
    mov esp, kernel_stack_top

    ret  ; return into kernel code

; Restore next process's context
restore_context:
    ; Load process's stack pointer
    mov esp, [next_proc + P_STACKFRAME + SP_OFF]

    ; Load process's page table (CR3)
    mov eax, [next_proc + P_CR3]
    mov cr3, eax            ; TLB flush happens here

    pop ebp
    pop edi
    pop esi
    pop edx
    pop ecx
    pop ebx
    pop eax

    iret  ; return to user mode (restores CS, EIP, EFLAGS, SS, ESP)
```

<br>

<h3 style="color: #E67E22;">Minix 3 Scheduler — Multi-Queue Priority</h3>

Unlike Linux's CFS (red-black tree), Minix 3 uses a simpler **multi-level queue** scheduler:

```c
/* kernel/proc.c (simplified) */
#define NR_SCHED_QUEUES 16   /* 16 priority levels */

struct proc *rdy_head[NR_SCHED_QUEUES]; /* head of each ready queue */
struct proc *rdy_tail[NR_SCHED_QUEUES]; /* tail of each ready queue */

/*
 * Priority levels (lower number = higher priority):
 *   0  : TASK (kernel tasks — clock, system)
 *   1-3: Drivers (disk, network, etc.)
 *   4-6: Servers (PM, VFS, VM, etc.)
 *   7-14: User processes
 *   15 : IDLE (only runs when nothing else can)
 */

void enqueue(struct proc *rp) {
    int q = rp->p_priority;  /* which queue */

    /* Add to tail of the appropriate queue */
    if (rdy_head[q] == NULL) {
        rdy_head[q] = rp;
    } else {
        rdy_tail[q]->p_nextready = rp;
    }
    rdy_tail[q] = rp;
    rp->p_nextready = NULL;
}

struct proc *pick_proc(void) {
    /* Find the highest-priority non-empty queue */
    for (int q = 0; q < NR_SCHED_QUEUES; q++) {
        if (rdy_head[q] != NULL) {
            return rdy_head[q];
        }
    }
    return idle_proc;  /* nothing to run */
}
```

Within each queue, processes get a **time quantum**. When the quantum expires, the process goes to the tail of its queue (round-robin within the same priority):

```c
/* kernel/clock.c (simplified) */
void clock_handler(void) {
    struct proc *p = get_cpulocal_var(proc_ptr); /* current process */

    p->p_ticks_left--;

    if (p->p_ticks_left <= 0) {
        /* Quantum expired — preempt */
        p->p_ticks_left = p->p_quantum_size;  /* reset quantum */
        dequeue(p);
        enqueue(p);    /* moves to tail of its priority queue */

        /* pick_proc() will run the next process in line */
    }
}
```

<br>

<h3 style="color: #E67E22;">Why Servers Get Higher Priority Than User Processes</h3>

```
Priority 0-1:  Kernel tasks (clock, system task)
Priority 2-4:  Device drivers (user-space, but high priority)
Priority 5-7:  System servers (PM, VFS, VM)
Priority 8-14: User processes
Priority 15:   IDLE

When a user process makes a syscall:
  1. User process (priority 8) sends message to VFS (priority 5)
  2. Scheduler picks VFS because it has higher priority
  3. VFS runs, processes the request, sends reply
  4. User process becomes runnable again
  5. Scheduler picks user process

This ensures system servers are responsive — they always preempt
user processes, so syscalls don't get stuck behind CPU-bound user tasks.
```

<br>

<h3 style="color: #E67E22;">Comparison: Linux CFS vs Minix 3 Multi-Queue</h3>

| Aspect | Linux CFS | Minix 3 Multi-Queue |
|--------|-----------|---------------------|
| Data structure | Red-black tree (O(log n)) | Array of linked lists (O(1)) |
| Fairness | Proportional fair sharing (vruntime) | Round-robin within priority level |
| Priority | Nice values adjust vruntime rate | Fixed priority queues |
| Complexity | ~5000 lines in `kernel/sched/fair.c` | ~200 lines in `kernel/proc.c` |
| Real-time support | Separate RT scheduler | No RT class (simpler use case) |
| Preemption granularity | Configurable (4ms default) | Fixed quantum per priority level |

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
8. "How do system calls work in a microkernel vs a monolithic kernel?"
9. "What is rendezvous IPC? How does Minix 3 use it?"
10. "Why are system servers given higher scheduling priority than user processes in Minix 3?"

---

**Next**: [Signals →](04-signals.md)
