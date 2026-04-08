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

## Interview Questions

1. "What are the main IPC mechanisms in Linux? Compare them."
2. "How do pipes work? What happens when the pipe buffer is full?"
3. "What is the difference between a pipe and a Unix domain socket?"
4. "When would you choose shared memory over sockets?"
5. "What synchronization is needed for shared memory?"
6. "How does the shell `|` operator work?"
7. "What is a named pipe (FIFO)? How is it different from an anonymous pipe?"
8. "How would you design a high-performance IPC mechanism for a trading system?"

---

**End of OS Appendix** | [← Back to OS Appendix](README.md)
