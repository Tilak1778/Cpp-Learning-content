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

## Interview Questions

1. "What is the difference between a process and a thread?"
2. "Explain `fork()`. What gets copied? What gets shared?"
3. "What is the `fork()` + `exec()` pattern? Why two separate calls?"
4. "What is a zombie process? How do you prevent them?"
5. "Why is creating a process more expensive than creating a thread?"
6. "How does Linux implement threads? What is `clone()`?"
7. "What happens to file descriptors after `fork()`?"

---

**Next**: [Context Switching & Scheduling →](03-context-switching.md)
