# OS Appendix 8: Pipes & IPC

[← Back to OS Appendix](README.md) | [← File System Internals](07-file-system.md)

---
---

# INTER-PROCESS COMMUNICATION (IPC)

---

<br>

<h2 style="color: #2980B9;">📘 8.1 Why IPC?</h2>

Processes have **isolated address spaces** (virtual memory). They cannot directly read or write each other's memory. IPC mechanisms are the ways processes communicate.

```
Process A                    Process B
┌──────────────┐             ┌──────────────┐
│ own memory   │             │ own memory   │
│ (isolated)   │             │ (isolated)   │
│              │    IPC      │              │
│  data ───────┼────────────►│ data         │
│              │  mechanism  │              │
└──────────────┘             └──────────────┘
```

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 8.2 Pipes — The Simplest IPC</h2>

A **pipe** is a unidirectional byte stream between two related processes (parent-child via `fork()`):

```cpp
#include <unistd.h>
#include <cstring>
#include <cstdio>

int main() {
    int pipefd[2];
    pipe(pipefd);
    //  pipefd[0] = read end
    //  pipefd[1] = write end

    pid_t pid = fork();

    if (pid == 0) {
        // Child — reads from pipe
        close(pipefd[1]);  // close write end (not needed)
        
        char buf[128];
        ssize_t n = read(pipefd[0], buf, sizeof(buf));
        buf[n] = '\0';
        printf("Child received: %s\n", buf);
        
        close(pipefd[0]);
    } else {
        // Parent — writes to pipe
        close(pipefd[0]);  // close read end (not needed)
        
        const char* msg = "Hello from parent!";
        write(pipefd[1], msg, strlen(msg));
        
        close(pipefd[1]);
        wait(nullptr);
    }
    return 0;
}
```

```
How pipes work internally:

  pipe() creates:
  ┌──────────────────────────────────────────┐
  │ Kernel buffer (typically 64KB on Linux)   │
  │                                           │
  │  write(fd[1]) ──► [data in buffer] ──► read(fd[0])  │
  │                                           │
  │  If buffer full: write() blocks            │
  │  If buffer empty: read() blocks            │
  └──────────────────────────────────────────┘

  After fork():
  Parent                     Child
  fd[1] ──► kernel buffer ──► fd[0]
  (write end)               (read end)
```

This is exactly what the shell `|` operator does:
```bash
ls -la | grep ".cpp" | wc -l

# Shell creates:
#   Process 1 (ls) → pipe → Process 2 (grep) → pipe → Process 3 (wc)
#   Each process's stdout is connected to the next process's stdin via pipes
```

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 8.3 Named Pipes (FIFOs) — Pipes for Unrelated Processes</h2>

Regular pipes only work between related processes (parent-child). **Named pipes (FIFOs)** have a filesystem path, so any process can open them:

```bash
# Terminal 1: Create and read from a FIFO
mkfifo /tmp/myfifo
cat /tmp/myfifo     # blocks until someone writes

# Terminal 2: Write to the FIFO
echo "Hello!" > /tmp/myfifo
```

```cpp
// Writer process
int fd = open("/tmp/myfifo", O_WRONLY);
write(fd, "data", 4);
close(fd);

// Reader process (different, unrelated process)
int fd = open("/tmp/myfifo", O_RDONLY);
char buf[128];
read(fd, buf, sizeof(buf));
close(fd);
```

FIFOs appear in the filesystem (`ls -l` shows type `p`) but are not stored on disk — they're kernel-buffered, just like anonymous pipes.

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 8.4 Unix Domain Sockets — Bidirectional Local IPC</h2>

Unix domain sockets provide **bidirectional**, **connection-oriented** (or datagram) communication between processes on the **same machine**:

```
Unix Domain Socket vs Internet Socket:

Internet Socket (TCP):
  Process A ──► network stack ──► NIC ──► wire ──► NIC ──► network stack ──► Process B
  (goes through full TCP/IP stack even if on same machine — slow)

Unix Domain Socket:
  Process A ──► kernel buffer ──► Process B
  (kernel copies directly — no network overhead, ~2x faster than TCP loopback)
```

```cpp
// Server (simplified)
int server_fd = socket(AF_UNIX, SOCK_STREAM, 0);

struct sockaddr_un addr;
addr.sun_family = AF_UNIX;
strcpy(addr.sun_path, "/tmp/my_socket");

bind(server_fd, (struct sockaddr*)&addr, sizeof(addr));
listen(server_fd, 5);

int client_fd = accept(server_fd, nullptr, nullptr);
// read/write with client_fd — full duplex

// Client (simplified)
int fd = socket(AF_UNIX, SOCK_STREAM, 0);

struct sockaddr_un addr;
addr.sun_family = AF_UNIX;
strcpy(addr.sun_path, "/tmp/my_socket");

connect(fd, (struct sockaddr*)&addr, sizeof(addr));
// read/write with fd — full duplex
```

Used by: Docker daemon, X11/Wayland, systemd, D-Bus, PostgreSQL local connections.

<br>

#### Bonus: Passing File Descriptors

A unique feature of Unix domain sockets — you can send **file descriptors** from one process to another via `sendmsg()`/`recvmsg()` with `SCM_RIGHTS` ancillary data. The receiving process gets a new fd pointing to the same open file description. This is how process managers hand off socket connections to worker processes.

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 8.5 Shared Memory — The Fastest IPC</h2>

Shared memory maps the **same physical frames** into multiple processes' virtual address spaces:

```
Process A's virtual space:         Process B's virtual space:
┌──────────────────┐               ┌──────────────────┐
│                  │               │                  │
│ shared region ───┼───────┐ ┌────┼── shared region  │
│                  │       │ │    │                  │
└──────────────────┘       ▼ ▼    └──────────────────┘
                     ┌────────────┐
                     │ Physical   │
                     │ Memory     │
                     │ (same      │
                     │  frames)   │
                     └────────────┘

No copying at all — both processes read/write the same memory.
But: YOU must synchronize access (mutexes, semaphores, atomics).
```

Two APIs for shared memory:

<br>

#### POSIX shared memory (`shm_open` + `mmap`)

```cpp
// Process A: Create and write
int fd = shm_open("/my_shm", O_CREAT | O_RDWR, 0666);
ftruncate(fd, 4096);
void* ptr = mmap(nullptr, 4096, PROT_READ | PROT_WRITE, MAP_SHARED, fd, 0);
close(fd);

strcpy((char*)ptr, "Hello from A!");

// Process B: Open and read
int fd = shm_open("/my_shm", O_RDONLY, 0);
void* ptr = mmap(nullptr, 4096, PROT_READ, MAP_SHARED, fd, 0);
close(fd);

printf("Read: %s\n", (char*)ptr);  // "Hello from A!"
```

#### mmap on the same file

```cpp
// Both processes:
int fd = open("/tmp/shared_data.bin", O_RDWR);
void* ptr = mmap(nullptr, 4096, PROT_READ | PROT_WRITE, MAP_SHARED, fd, 0);
// ptr points to the same physical pages in both processes
```

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 8.6 Message Queues</h2>

POSIX message queues provide **structured, prioritized messages** between processes:

```cpp
#include <mqueue.h>

// Sender
mqd_t mq = mq_open("/my_queue", O_WRONLY | O_CREAT, 0666, nullptr);
mq_send(mq, "Hello", 5, 0);  // priority 0
mq_close(mq);

// Receiver
mqd_t mq = mq_open("/my_queue", O_RDONLY);
char buf[256];
unsigned int priority;
mq_receive(mq, buf, sizeof(buf), &priority);
printf("Received: %s (priority %u)\n", buf, priority);
mq_close(mq);
```

Unlike pipes, message queues preserve **message boundaries** (you get whole messages, not arbitrary byte chunks).

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 8.7 Comparison of IPC Methods</h2>

| Method | Direction | Speed | Boundary | Related? | Best For |
|--------|-----------|-------|----------|----------|----------|
| **Pipe** | Unidirectional | Fast | Byte stream | Parent-child | Simple data flow |
| **Named Pipe (FIFO)** | Unidirectional | Fast | Byte stream | Any | Simple, unrelated processes |
| **Unix Domain Socket** | Bidirectional | Very fast | Stream or datagram | Any | Client-server on same machine |
| **TCP Socket (loopback)** | Bidirectional | Slower | Stream | Any | Network-transparent |
| **Shared Memory** | Bidirectional | Fastest | None (raw bytes) | Any | High-throughput, low-latency |
| **Message Queue** | Bidirectional | Moderate | Message boundaries | Any | Structured messages with priority |
| **Signals** | Unidirectional | Fast | None (just a number) | Any (by PID) | Simple notifications |

<br>

#### Decision Flowchart

```
Need to communicate between processes?
  │
  ├── Same machine?
  │     ├── Need highest performance? → Shared Memory + atomics/mutex
  │     ├── Need request/response? → Unix Domain Socket
  │     ├── Simple parent→child data flow? → Pipe
  │     ├── Simple unrelated data flow? → Named Pipe (FIFO)
  │     └── Need message boundaries + priority? → Message Queue
  │
  └── Different machines?
        └── TCP/UDP Sockets (covered in main study plan)
```

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 8.8 Common IPC Patterns in C++ Systems</h2>

| Pattern | IPC Used | Example |
|---------|----------|---------|
| **Worker pool** | Unix domain socket or pipe | Web server forks workers, parent distributes connections |
| **Logging daemon** | Unix domain socket | Application sends log messages to a central logger (syslog) |
| **Database connection** | Unix domain socket | PostgreSQL uses `/var/run/postgresql/.s.PGSQL.5432` |
| **Shared cache** | Shared memory | Multiple processes sharing a read-heavy cache (Redis-like) |
| **Process supervision** | Signals + pipe | systemd monitors child processes with `SIGCHLD` and status pipes |
| **Plugin hot-reload** | Shared memory + signals | Send `SIGHUP` → process re-reads config from shared memory |

<br><br>

---

## Minix 3 Deep Dive — Message Passing as THE IPC Mechanism

In Minix 3, message passing isn't just one IPC option — it's the **foundational mechanism** that the entire OS is built on. Every system call, every driver request, every file operation is a message. Understanding Minix 3's IPC is understanding the microkernel itself.

<br>

<h3 style="color: #E67E22;">Source Files for IPC</h3>

```
minix/kernel/
├── proc.c              ← mini_send(), mini_receive(), mini_notify()
│                          THE core IPC implementation (~500 lines)
├── proc.h              ← process state flags (SENDING, RECEIVING, etc.)
└── system/
    └── do_copy.c       ← inter-process memory copy (for large data)

minix/servers/vfs/
├── pipe.c              ← pipe() implementation (uses VFS + kernel IPC)

minix/include/minix/
├── ipc.h               ← message structure definition
├── com.h               ← message type constants (VFS_READ, PM_FORK, etc.)
└── syslib.h            ← user-space IPC wrappers (send, receive, sendrec)

minix/lib/libsys/
├── sys_fork.c          ← wrapper for SYS_FORK kernel call
├── sys_kill.c          ← wrapper for SYS_KILL
└── (one file per kernel call)
```

<br>

<h3 style="color: #E67E22;">The Three IPC Primitives</h3>

Minix 3's kernel provides exactly three IPC operations:

```c
/* The fundamental IPC primitives (kernel/proc.c) */

/* 1. SEND — send a message to a process (blocks until delivered) */
int mini_send(struct proc *caller, endpoint_t dst, message *msg, int flags);

/* 2. RECEIVE — wait for a message (blocks until one arrives) */
int mini_receive(struct proc *caller, endpoint_t src, message *msg, int flags);

/* 3. NOTIFY — non-blocking one-bit notification (like a lightweight signal) */
int mini_notify(struct proc *caller, endpoint_t dst);

/* Convenience combinations: */
/* SENDREC = send + receive reply (the common syscall pattern) */
/* SENDNB = non-blocking send (fails immediately if can't deliver) */
```

<br>

<h3 style="color: #E67E22;">mini_send() — The Full Implementation</h3>

```c
/* kernel/proc.c (simplified but complete logic) */
int mini_send(struct proc *caller, endpoint_t dst_e, message *m_ptr, int flags) {
    struct proc *dst_ptr;
    int dst_p;

    /* Convert endpoint to process number */
    if (!isokendpt(dst_e, &dst_p))
        return EDEADSRCDST;
    dst_ptr = proc_addr(dst_p);

    /* Check if destination exists and is alive */
    if (RTS_ISSET(dst_ptr, RTS_NO_ENDPOINT))
        return EDEADSRCDST;

    /* DEADLOCK DETECTION: check if sending would create a cycle */
    /* A→B→C→A would mean everyone blocks forever */
    if (flags & NON_BLOCKING) {
        /* skip deadlock check for non-blocking */
    } else {
        struct proc *xp = dst_ptr;
        while (xp && RTS_ISSET(xp, RTS_SENDING)) {
            xp = proc_addr(xp->p_sendto);
            if (xp == caller) return ELOCKED;  /* cycle detected! */
        }
    }

    /* Is the destination waiting to receive from us (or from ANY)? */
    if (RTS_ISSET(dst_ptr, RTS_RECEIVING) &&
        (dst_ptr->p_getfrom_e == caller->p_endpoint ||
         dst_ptr->p_getfrom_e == ANY)) {

        /* Destination IS waiting — deliver the message directly */
        /* Copy message from sender's address space to receiver's */
        copy_msg(caller, m_ptr, dst_ptr, dst_ptr->p_messbuf);

        dst_ptr->p_messbuf->m_source = caller->p_endpoint;

        /* Unblock the receiver */
        RTS_UNSET(dst_ptr, RTS_RECEIVING);
        /* dst_ptr is now runnable again */

    } else {
        /* Destination is NOT waiting — sender must block */

        if (flags & NON_BLOCKING)
            return ENOTREADY;

        /* Block the sender */
        RTS_SET(caller, RTS_SENDING);
        caller->p_sendto_e = dst_e;
        caller->p_messbuf = m_ptr;

        /* Add caller to destination's send queue (linked list) */
        struct proc **xpp = &dst_ptr->p_caller_q;
        while (*xpp) xpp = &(*xpp)->p_q_link;
        *xpp = caller;
        caller->p_q_link = NULL;
    }

    return OK;
}
```

<br>

<h3 style="color: #E67E22;">mini_receive() — The Full Implementation</h3>

```c
/* kernel/proc.c (simplified but complete logic) */
int mini_receive(struct proc *caller, endpoint_t src_e,
                 message *m_ptr, int flags) {

    /* Check if there's a notification pending (asynchronous event) */
    if (caller->p_ntf_pending) {
        /* Build a notification message */
        build_notify_msg(caller, m_ptr);
        return OK;
    }

    /* Check the sender queue — is anyone already waiting to send to us? */
    struct proc *sender = NULL;

    if (src_e == ANY) {
        /* Accept from anyone — take the first sender in queue */
        sender = caller->p_caller_q;
    } else {
        /* Only accept from a specific sender */
        struct proc **xpp = &caller->p_caller_q;
        while (*xpp) {
            if ((*xpp)->p_endpoint == src_e) {
                sender = *xpp;
                *xpp = sender->p_q_link;  /* remove from queue */
                break;
            }
            xpp = &(*xpp)->p_q_link;
        }
    }

    if (sender) {
        /* Found a waiting sender — grab their message */
        copy_msg(sender, sender->p_messbuf, caller, m_ptr);
        m_ptr->m_source = sender->p_endpoint;

        /* Unblock the sender */
        RTS_UNSET(sender, RTS_SENDING);

        if (src_e == ANY) {
            /* Remove sender from head of queue */
            caller->p_caller_q = sender->p_q_link;
        }

    } else {
        /* Nobody waiting — receiver must block */

        if (flags & NON_BLOCKING)
            return ENOTREADY;

        RTS_SET(caller, RTS_RECEIVING);
        caller->p_getfrom_e = src_e;
        caller->p_messbuf = m_ptr;
    }

    return OK;
}
```

<br>

<h3 style="color: #E67E22;">How a System Call Works — End to End</h3>

```c
/* Example: User process calls read(fd, buf, 100) */

/* Step 1: libc packs the message */
/* lib/libc/sys/read.c */
ssize_t read(int fd, void *buf, size_t count) {
    message m;
    m.m_type = VFS_READ;
    m.m_lc_vfs_readwrite.fd = fd;
    m.m_lc_vfs_readwrite.buf = buf;
    m.m_lc_vfs_readwrite.len = count;

    /* SENDREC = send to VFS and wait for reply */
    _syscall(VFS_PROC_NR, VFS_READ, &m);

    return m.m_type;  /* reply contains bytes read (or error) */
}

/* Step 2: _syscall does a kernel trap */
/* Enters kernel, calls mini_send(caller, VFS, &m) */
/* Then calls mini_receive(caller, VFS, &reply) */

/* Step 3: VFS server receives the message in its main loop */
/* servers/vfs/main.c */
void main(void) {
    message m;
    while (TRUE) {
        receive(ANY, &m);     /* wait for any message */

        int who = m.m_source; /* who sent it */
        int call = m.m_type;  /* which syscall */

        /* Dispatch to handler */
        int result = call_vec[call](&m);

        /* Send reply back */
        m.m_type = result;
        send(who, &m);
    }
}

/* Step 4: VFS handler for read */
/* servers/vfs/read.c → talks to MFS via another send/receive */
/* MFS reads data from disk, replies to VFS */
/* VFS replies to user process */
/* User process unblocks, read() returns */
```

<br>

<h3 style="color: #E67E22;">Notifications — Asynchronous Lightweight Messages</h3>

`mini_notify()` is different from send/receive — it's a **non-blocking, bitmap-based** mechanism for asynchronous events:

```c
/* kernel/proc.c (simplified) */
int mini_notify(struct proc *caller, endpoint_t dst_e) {
    struct proc *dst_ptr = proc_addr(dst_e);

    /* Set a bit in the destination's pending notification bitmap */
    set_bit(dst_ptr->p_ntf_pending, caller->p_nr);

    /* If destination is blocked in receive(), unblock it */
    if (RTS_ISSET(dst_ptr, RTS_RECEIVING)) {
        RTS_UNSET(dst_ptr, RTS_RECEIVING);
    }

    /* Sender NEVER blocks — always returns immediately */
    return OK;
}
```

Notifications are used for:
- Hardware interrupts (driver gets notified when device is ready)
- Timer events (clock tick notifications)
- Kernel alerts to servers (e.g., "a process has a pending signal")

<br>

<h3 style="color: #E67E22;">Pipes in Minix 3 — Built on VFS + Shared Buffer</h3>

Pipes in Minix 3 are implemented in the VFS server, NOT in the kernel:

```c
/* servers/vfs/pipe.c (simplified) */
int do_pipe(void) {
    /* Allocate a vnode (virtual inode) for the pipe */
    struct vnode *vp = get_free_vnode();
    vp->v_pipe = TRUE;
    vp->v_size = 0;     /* pipe buffer starts empty */

    /* Allocate pipe buffer (in VFS memory) */
    vp->v_pipe_buf = alloc_pipe_buf(PIPE_BUF_SIZE);

    /* Create two file descriptions — read end and write end */
    struct filp *fil_rd = get_free_filp();
    fil_rd->filp_vno = vp;
    fil_rd->filp_mode = R_BIT;

    struct filp *fil_wr = get_free_filp();
    fil_wr->filp_vno = vp;
    fil_wr->filp_mode = W_BIT;

    /* Allocate two file descriptors for the calling process */
    int fd_rd = get_free_fd(fp);  /* fd[0] = read end */
    int fd_wr = get_free_fd(fp);  /* fd[1] = write end */

    fp->fp_filp[fd_rd] = fil_rd;
    fp->fp_filp[fd_wr] = fil_wr;

    /* Return both fds to the caller */
    m_out.pipefd[0] = fd_rd;
    m_out.pipefd[1] = fd_wr;

    return OK;
}
```

Writing to a pipe:
```c
/* servers/vfs/pipe.c — pipe write (simplified) */
int pipe_write(struct filp *fil, char *buf, size_t count) {
    struct vnode *vp = fil->filp_vno;

    if (pipe_buffer_full(vp)) {
        /* Buffer full — suspend the writer */
        suspend_process(fp, FP_BLOCKED_ON_PIPE);
        /* Writer will be woken when reader reads some data */
        return SUSPEND;
    }

    /* Copy data into pipe buffer */
    size_t written = copy_to_pipe_buf(vp, buf, count);
    vp->v_size += written;

    /* Wake up any blocked readers */
    wakeup_readers(vp);

    return written;
}
```

<br>

<h3 style="color: #E67E22;">Deadlock Prevention in Minix 3 IPC</h3>

A critical problem with synchronous message passing: if A sends to B and B sends to A simultaneously, both block forever. Minix 3 detects this:

```c
/* Deadlock detection in mini_send (simplified) */

/* Follow the chain of "who is X trying to send to?" */
struct proc *xp = dst_ptr;
while (RTS_ISSET(xp, RTS_SENDING)) {
    xp = proc_addr(xp->p_sendto);
    if (xp == caller) {
        /* Cycle found: caller → ... → dst → ... → caller */
        return ELOCKED;
    }
}

/*
Example cycle:
  PM wants to send to VFS (PM blocked, SENDING)
  VFS wants to send to PM (would create PM→VFS→PM cycle)
  → ELOCKED returned to VFS

This is why Minix 3 has the SENDREC primitive:
  SENDREC = send + immediately receive
  This prevents the common deadlock pattern where two servers
  try to send to each other simultaneously.
*/
```

<br>

<h3 style="color: #E67E22;">Comparison: Linux IPC vs Minix 3 Kernel IPC</h3>

| Aspect | Linux User-Space IPC | Minix 3 Kernel IPC |
|--------|---------------------|-------------------|
| Purpose | Optional — processes CAN use IPC | Mandatory — the OS IS IPC |
| Performance | Variable (pipes ~100μs, shared mem ~ns) | ~5-10μs per message |
| Reliability | Process crashes don't affect kernel | Server crashes are recoverable |
| Deadlock | Programmer's problem | Kernel detects and prevents |
| Scalability | Many mechanisms for different needs | One mechanism for everything |
| Buffer | Kernel-managed buffers (pipes: 64KB) | Fixed-size messages (~56 bytes) |

<br>

<h3 style="color: #E67E22;">The Reincarnation Server — Self-Healing IPC</h3>

One of Minix 3's most innovative features is the **Reincarnation Server (RS)** — it monitors all servers and drivers via IPC:

```
RS periodically sends "are you alive?" notification to each server:
  RS ──notify──► PM     PM ──notify──► RS (I'm alive!)
  RS ──notify──► VFS    VFS ──notify──► RS (I'm alive!)
  RS ──notify──► VM     VM ──notify──► RS (I'm alive!)

If a server crashes or doesn't respond:
  RS detects the failure
  RS kills the crashed server
  RS starts a fresh copy of the server
  RS sends "take over" messages to the new server
  The new server reconstructs its state from surviving servers

This means: a disk driver crash → RS restarts the driver → system continues
In Linux: a disk driver crash → kernel panic → reboot
```

This self-healing capability is only possible because IPC is the fundamental communication mechanism — replacing a server is just a matter of routing messages to a new process.

<br><br>

---

## Interview Questions

1. "What are the main IPC mechanisms in Linux? Compare them."
2. "How do pipes work? What happens when the pipe buffer is full?"
3. "What is the difference between a pipe and a Unix domain socket?"
4. "When would you choose shared memory over sockets?"
5. "What synchronization is needed for shared memory?"
6. "How does the shell `|` operator work?"
7. "What is a named pipe (FIFO)? How is it different from an anonymous pipe?"
8. "How would you design a high-performance IPC mechanism for a trading system?"
9. "Explain Minix 3's synchronous message passing: send, receive, and notify."
10. "How does Minix 3 detect IPC deadlocks?"
11. "What is the Reincarnation Server and why is it important?"
12. "What are the trade-offs of building an entire OS on synchronous message passing?"

---

**End of OS Appendix** | [← Back to OS Appendix](README.md)
