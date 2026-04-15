# OS Appendix 2: Process Model

[← Back to OS Appendix](README.md) | [← Virtual Memory](01-virtual-memory.md)

---
---

# PROCESSES, FORK, EXEC & THREADS

---

<br>

<h2 style="color: #2980B9;">📘 2.1 What Is a Process?</h2>

A process is a **running instance of a program**. The OS gives each process:

| Resource | Description |
|----------|-------------|
| **Address space** | Its own virtual memory (text, data, heap, stack) |
| **Registers** | Program counter, stack pointer, general-purpose registers |
| **File descriptor table** | Open files, sockets, pipes |
| **PID** | Unique process ID |
| **Credentials** | User ID, group ID |
| **Signal handlers** | How to handle signals |
| **Environment** | Environment variables |

```
Process = Code + Data + Resources + Execution State
```

Each process is **isolated** — one process cannot directly access another's memory (virtual memory enforces this).

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 2.2 Process vs Thread</h2>

```
Process:                              Thread:
┌─────────────────────┐               ┌─────────────────────┐
│ Own address space    │               │ SHARED address space │
│ Own file descriptors │               │ SHARED file desc.    │
│ Own page table       │               │ SHARED page table    │
│ Expensive to create  │               │ Cheap to create      │
│ Isolated (safe)      │               │ Shared (fast but     │
│                      │               │   need synchronization)│
│ IPC needed to        │               │ Direct memory access │
│  communicate         │               │  between threads     │
│                      │               │                      │
│ ┌──── Thread 1 ────┐ │               │ ┌──── Thread 1 ────┐ │
│ │ Own stack         │ │               │ │ Own stack         │ │
│ │ Own registers     │ │               │ │ Own registers     │ │
│ └───────────────────┘ │               │ ├──── Thread 2 ────┤ │
└─────────────────────┘               │ │ Own stack         │ │
                                      │ │ Own registers     │ │
                                      │ └───────────────────┘ │
                                      └─────────────────────┘
```

**Key differences:**

| | Process | Thread |
|---|---------|--------|
| Memory | Separate address spaces | Shared address space |
| Creation cost | Expensive (~ms, copy page tables) | Cheap (~μs, just new stack) |
| Communication | IPC (pipes, sockets, shared mem) | Direct shared memory |
| Crash isolation | One crash doesn't kill others | One crash kills ALL threads |
| Switching cost | Expensive (TLB flush, page table swap) | Cheaper (same address space) |

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 2.3 <code>fork()</code> — Creating a Child Process</h2>

`fork()` creates a **copy** of the current process:

```cpp
#include <unistd.h>
#include <iostream>

int main() {
    std::cout << "Before fork, PID: " << getpid() << "\n";

    pid_t pid = fork();

    if (pid < 0) {
        // Error
        std::cerr << "fork failed\n";
    } else if (pid == 0) {
        // CHILD process — fork() returned 0
        std::cout << "Child PID: " << getpid()
                  << ", Parent PID: " << getppid() << "\n";
    } else {
        // PARENT process — fork() returned child's PID
        std::cout << "Parent PID: " << getpid()
                  << ", Child PID: " << pid << "\n";
    }
    return 0;
}
```

**After fork, two processes execute from the same point:**

```
Before fork():
  One process running
                │
          fork() called
               ╱╲
              ╱  ╲
             ╱    ╲
  Parent (pid>0)  Child (pid==0)
  continues       continues
  same code       same code
```

What gets copied:
- Entire address space (via **COW** — cheap, see Appendix 1)
- File descriptor table (both share the same open files)
- Signal handlers
- Environment variables

What's different:
- PID (child gets a new one)
- Return value of `fork()` (parent gets child's PID, child gets 0)
- Parent PID (`getppid()`)

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 2.4 <code>exec()</code> — Replacing the Process Image</h2>

`exec()` **replaces** the current process's code, data, heap, and stack with a new program. The PID stays the same.

```cpp
#include <unistd.h>

int main() {
    pid_t pid = fork();

    if (pid == 0) {
        // Child: replace itself with "ls -la"
        execlp("ls", "ls", "-la", nullptr);
        // If exec succeeds, this line NEVER executes
        perror("exec failed");
    } else {
        // Parent: wait for child
        wait(nullptr);
        std::cout << "Child finished\n";
    }
    return 0;
}
```

```
fork() + exec() = the standard way to launch a new program:

Parent process
     │
   fork()
     ├──────── Child process
     │              │
     │          exec("ls")
     │              │
     │         ┌────┴────────────────┐
     │         │ ls program runs     │
     │         │ (same PID, new code)│
     │         └────┬────────────────┘
     │              │ exit
   wait()           │
     │◄─────────────┘
     │
  continues
```

The `exec` family: `execl`, `execlp`, `execle`, `execv`, `execvp`, `execvpe` — they differ in how arguments and environment are passed, but all replace the process image.

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 2.5 <code>wait()</code> — Reaping Child Processes</h2>

When a child process exits, it becomes a **zombie** — it's dead but its exit status remains in the kernel until the parent calls `wait()` or `waitpid()`:

```
Child exits → becomes zombie (shown as <defunct> in ps)
Parent calls wait() → kernel gives exit status, zombie removed
```

If the parent never calls `wait()`, zombies accumulate (resource leak).

If the parent exits before the child, the child becomes an **orphan** and is adopted by PID 1 (`init`/`systemd`), which will reap it.

```cpp
#include <sys/wait.h>

pid_t pid = fork();
if (pid == 0) {
    // Child
    exit(42);
}

// Parent
int status;
waitpid(pid, &status, 0);  // blocks until child exits
if (WIFEXITED(status)) {
    std::cout << "Child exited with code: " << WEXITSTATUS(status) << "\n";
    // prints: Child exited with code: 42
}
```

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 2.6 Process States</h2>

```
                ┌─────────────┐
                │   Created   │
                └──────┬──────┘
                       │ admitted
                       ▼
                ┌─────────────┐    interrupt    ┌──────────────┐
         ┌─────│    Ready     │◄───────────────│   Running     │
         │     └──────┬──────┘                 └───┬───┬──────┘
         │            │ scheduler dispatch         │   │
         │            └───────────────────────────►│   │
         │                                         │   │ I/O or event wait
         │                                         │   ▼
         │                                    exit │  ┌──────────────┐
         │                                         │  │   Blocked    │
         │                                         │  │  (Waiting)   │
         │                                         │  └──────┬───────┘
         │                                         │         │ I/O done
         │                                         │         ▼
         │                                         │    (goes to Ready)
         │                                         ▼
         │                                   ┌──────────────┐
         └──────────────────────────────────►│  Terminated   │
                                             │  (Zombie)     │
                                             └──────────────┘
```

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 2.7 Linux Thread Implementation</h2>

On Linux, threads are implemented using the `clone()` system call — they're essentially processes that **share** address space, file descriptors, and signal handlers:

```
clone() flags for a thread:
  CLONE_VM        — share address space
  CLONE_FS        — share filesystem info
  CLONE_FILES     — share file descriptor table
  CLONE_SIGHAND   — share signal handlers
  CLONE_THREAD    — same thread group (same PID to outside world)
```

This is why Linux calls them "lightweight processes" or "tasks" — there's no fundamental difference between a process and a thread at the kernel level. The difference is just **which resources are shared**.

`pthread_create()` and `std::thread` both use `clone()` under the hood on Linux.

<br><br>

---

## Minix 3 Deep Dive — Process Management

Minix 3's microkernel design separates process management into **three distinct layers**, making it much clearer to see what each part does.

<br>

<h3 style="color: #E67E22;">The Three Layers of Process Management</h3>

```
Layer 1: Kernel (minix/kernel/)
  - Maintains the process table with scheduling state
  - Handles context switching (save/restore registers)
  - Does NOT manage fork, exec, or wait — delegates to servers

Layer 2: Process Manager (minix/servers/pm/)
  - Handles fork(), exec(), exit(), wait(), signal delivery
  - Maintains its own process table (PID, parent, exit status, signal state)
  - Communicates with kernel and VM server via messages

Layer 3: VM Server (minix/servers/vm/)
  - Handles address space creation/duplication
  - Called by PM during fork() and exec()

Linux does ALL of this inside the kernel in kernel/fork.c, kernel/exec.c, kernel/exit.c
```

<br>

<h3 style="color: #E67E22;">Minix 3 Source Tree — Process Management</h3>

```
minix/servers/pm/
├── main.c       ← PM server main loop (message receive → dispatch)
├── table.c      ← system call dispatch table (maps call numbers to handlers)
├── forkexit.c   ← fork(), exit(), wait() implementation
├── exec.c       ← exec() implementation (coordinates with VFS + VM)
├── signal.c     ← signal delivery and handling
├── mproc.h      ← process table entry structure (PM's view)
├── getset.c     ← getpid, setuid, etc.
└── misc.c       ← miscellaneous syscalls (brk, reboot, etc.)

minix/kernel/
├── proc.h       ← kernel process table entry (scheduling view)
├── proc.c       ← scheduling, IPC (send/receive), context switch
├── system/
│   ├── do_fork.c    ← kernel-side fork (copy kernel slot)
│   ├── do_exec.c    ← kernel-side exec (reset kernel state)
│   ├── do_exit.c    ← kernel-side exit (cleanup kernel slot)
│   └── do_newmap.c  ← set up memory map for a process
└── table.c      ← kernel system call dispatch table
```

<br>

<h3 style="color: #E67E22;">The Process Table — Three Copies</h3>

In Minix 3, there isn't one process table — there are **three**, each owned by a different component:

```c
/* kernel/proc.h — kernel's process table entry */
struct proc {
    struct stackframe_s p_reg;     /* saved registers */
    reg_t          p_ldt_sel;      /* LDT selector */
    proc_nr_t      p_nr;          /* process slot number */
    char           p_priority;     /* scheduling priority */
    char           p_quantum_size; /* time quantum */
    int            p_ticks_left;   /* ticks remaining in quantum */
    struct proc   *p_nextready;    /* next in ready queue */
    u32_t          p_rts_flags;    /* SLOT_FREE, SENDING, RECEIVING, etc. */
    /* ... */
};

/* servers/pm/mproc.h — PM's process table entry */
struct mproc {
    pid_t          mp_pid;         /* process ID */
    pid_t          mp_parent;      /* parent's PID */
    int            mp_exitstatus;  /* exit status */
    sigset_t       mp_sigpending;  /* pending signals */
    sigset_t       mp_sigmask;     /* blocked signals */
    struct sigaction mp_sigact[_NSIG]; /* signal handlers */
    uid_t          mp_realuid;     /* real user ID */
    gid_t          mp_realgid;     /* real group ID */
    unsigned       mp_flags;       /* IN_USE, WAITING, ZOMBIE, etc. */
    /* ... */
};

/* servers/vfs/fproc.h — VFS's process table entry */
struct fproc {
    struct filp   *fp_filp[OPEN_MAX]; /* open file descriptors */
    dev_t          fp_tty;            /* controlling terminal */
    ino_t          fp_rootdir;        /* root directory inode */
    ino_t          fp_workdir;        /* working directory inode */
    mode_t         fp_umask;          /* file creation mask */
    /* ... */
};
```

This separation is key to understanding microkernels: each server only knows what it needs to know. The kernel knows registers and scheduling. PM knows PIDs and signals. VFS knows file descriptors.

<br>

<h3 style="color: #E67E22;">How fork() Works in Minix 3 — The Message Flow</h3>

In Linux, `fork()` is a single kernel function. In Minix 3, it involves **four components** communicating via messages:

```
User Process          PM Server            VM Server           Kernel
    │                    │                     │                  │
    │──fork() syscall──►│                     │                  │
    │                    │                     │                  │
    │                    │──SYS_FORK─────────────────────────────►│
    │                    │                     │    Kernel copies  │
    │                    │                     │    kernel proc    │
    │                    │◄─────────────────────────────OK─────────│
    │                    │                     │                  │
    │                    │──VM_FORK──────────►│                  │
    │                    │                     │ VM duplicates     │
    │                    │                     │ address space     │
    │                    │                     │ (COW setup)       │
    │                    │◄───────────OK───────│                  │
    │                    │                     │                  │
    │                    │ PM copies its own   │                  │
    │                    │ mproc entry         │                  │
    │                    │                     │                  │
    │                    │──VFS_FORK (to VFS)  │                  │
    │                    │ VFS copies fproc    │                  │
    │                    │ (file descriptors)  │                  │
    │                    │                     │                  │
    │◄──returns child_pid│                     │                  │
```

The actual fork code in `servers/pm/forkexit.c`:

```c
/* servers/pm/forkexit.c (simplified) */
int do_fork(void) {
    struct mproc *parent = mp;  /* current process (caller) */
    int child_slot;

    /* Find a free slot in PM's process table */
    for (child_slot = 0; child_slot < NR_PROCS; child_slot++) {
        if (mproc[child_slot].mp_flags & IN_USE) continue;
        break;
    }

    struct mproc *child = &mproc[child_slot];

    /* Copy PM's process table entry */
    *child = *parent;
    child->mp_pid = get_next_pid();
    child->mp_parent = parent->mp_pid;
    child->mp_sigpending = 0;   /* child starts with no pending signals */

    /* Tell the kernel to copy its process slot */
    sys_fork(parent->mp_endpoint, child_slot, child->mp_pid, ...);

    /* Tell VM server to duplicate the address space (COW) */
    vm_fork(parent->mp_endpoint, child_slot, child->mp_pid);

    /* Tell VFS to copy file descriptor table */
    tell_vfs(child, parent);

    /* Return child's PID to parent, 0 to child */
    reply(parent->mp_endpoint, child->mp_pid);
    reply(child->mp_endpoint, 0);

    return OK;
}
```

<br>

<h3 style="color: #E67E22;">How exec() Works in Minix 3</h3>

`exec()` is even more interesting in a microkernel — it involves PM, VFS, and VM cooperating:

```
1. PM receives exec() syscall
2. PM asks VFS to read the executable file headers (ELF parsing)
3. VFS reads the file, extracts segment info (text, data, bss sizes)
4. PM asks VM to destroy old address space and create new one
5. VM sets up new regions: text (from file), data (from file), bss (zeroed), stack
6. PM asks VFS to load the actual segments into the new address space
7. PM tells kernel to reset the process's registers (new entry point, stack pointer)
8. Process resumes at the new program's entry point
```

```c
/* servers/pm/exec.c (simplified flow) */
int do_exec(void) {
    /* Step 1: Ask VFS to open and parse the executable */
    tell_vfs(VFS_EXEC_OPEN, pathname, ...);
    /* VFS reads ELF headers, returns: text_size, data_size, entry_point */

    /* Step 2: Ask VM to create new address space */
    vm_exec_newmem(mp->mp_endpoint,
                   text_size, data_size, bss_size,
                   stack_size, entry_point, ...);
    /* VM destroys old regions, creates new ones */

    /* Step 3: Ask VFS to load segments into the new space */
    tell_vfs(VFS_EXEC_LOAD, ...);
    /* VFS reads file data into the process's new memory */

    /* Step 4: Tell kernel to set up registers */
    sys_exec(mp->mp_endpoint, new_stack_ptr, entry_point, ...);
    /* Kernel sets PC = entry_point, SP = stack top */

    /* Process now runs the new program */
    return OK;
}
```

<br>

<h3 style="color: #E67E22;">Zombie Reaping in Minix 3</h3>

When a process exits, PM handles the zombie state. The child becomes a zombie in PM's table until the parent calls `wait()`:

```c
/* servers/pm/forkexit.c — exit handling (simplified) */
int do_exit(int exit_status) {
    struct mproc *dying = mp;

    /* Tell VM to free the address space */
    vm_exit(dying->mp_endpoint);

    /* Tell VFS to close all file descriptors */
    tell_vfs(VFS_EXIT, dying->mp_endpoint);

    /* Tell kernel to stop scheduling this process */
    sys_exit(dying->mp_endpoint);

    /* Mark as zombie — entry preserved for parent's wait() */
    dying->mp_flags |= ZOMBIE;
    dying->mp_exitstatus = exit_status;

    /* Notify parent (if parent is in wait()) */
    struct mproc *parent = &mproc[dying->mp_parent];
    if (parent->mp_flags & WAITING) {
        /* Parent is blocked in wait() — deliver the result now */
        cleanup_zombie(dying);
        reply(parent->mp_endpoint, dying->mp_pid);
    }

    /* If parent already exited, init (PID 1) adopts and reaps */
    if (!(parent->mp_flags & IN_USE)) {
        dying->mp_parent = INIT_PID;
        cleanup_zombie(dying);
    }
}
```

<br>

<h3 style="color: #E67E22;">Microkernel vs Monolithic — Process Model Comparison</h3>

| Aspect | Linux (Monolithic) | Minix 3 (Microkernel) |
|--------|-------------------|----------------------|
| fork() | Single `kernel/fork.c` function | PM + VM + VFS + kernel cooperation via messages |
| Process table | One unified `task_struct` | Three separate tables (kernel, PM, VFS) |
| exec() | `fs/exec.c` in kernel | PM coordinates VFS (file I/O) + VM (address space) |
| Performance | Fast (no IPC overhead) | Slower (~3-5x for fork/exec due to message passing) |
| Reliability | One bug in fork → kernel panic | PM crash → PM can be restarted by reincarnation server |
| Code clarity | Complex (`task_struct` is ~600 fields) | Clean separation of concerns |

The Minix 3 approach shows you exactly **what information each part of the OS needs** about a process — something that's hard to see when everything is tangled together in Linux's `task_struct`.

<br><br>

---

## Interview Questions

1. "What is the difference between a process and a thread?"
2. "Explain `fork()`. What gets copied? What gets shared?"
3. "What is the `fork()` + `exec()` pattern? Why two separate calls?"
4. "What is a zombie process? How do you prevent them?"
5. "Why is creating a process more expensive than creating a thread?"
6. "How does Linux implement threads? What is `clone()`?"
7. "What happens to file descriptors after `fork()`?"
8. "In a microkernel, how is the process table split across components?"
9. "Walk through the message flow of fork() in Minix 3."

---

**Next**: [Context Switching & Scheduling →](03-context-switching.md)
