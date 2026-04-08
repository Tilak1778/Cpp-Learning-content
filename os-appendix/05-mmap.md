# OS Appendix 5: Memory-Mapped I/O

[← Back to OS Appendix](README.md) | [← Signals](04-signals.md)

---
---

# MMAP — MEMORY-MAPPED FILES AND I/O

---

<br>

<h2 style="color: #2980B9;">📘 5.1 The Problem with <code>read()</code> / <code>write()</code></h2>

Traditional file I/O involves **two copies** of the data:

```
Reading a file with read():

  Disk → [Kernel page cache buffer] → [User-space buffer]
                copy 1                     copy 2

  1. Kernel reads from disk into its page cache (if not already cached)
  2. Kernel copies from page cache to your user-space buffer
  3. You process the data in your buffer
```

For large files or frequent access, this double-copy is wasteful. What if your process could directly access the kernel's page cache?

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 5.2 What Is <code>mmap()</code>?</h2>

`mmap()` maps a file (or anonymous memory) directly into your process's virtual address space:

```
After mmap():

  Your virtual address space:
  ┌──────────────────────┐
  │  ...                 │
  │  Stack               │
  │  ...                 │
  │  mmap'd region ──────┼──► Points directly to kernel page cache
  │  [file contents      │    (NO copy into user-space buffer)
  │   visible here]      │
  │  ...                 │
  │  Heap                │
  │  .text               │
  └──────────────────────┘

  Accessing the memory = accessing the file
  No read()/write() syscalls needed!
```

```cpp
#include <sys/mman.h>
#include <fcntl.h>
#include <unistd.h>
#include <cstdio>

int main() {
    int fd = open("data.bin", O_RDWR);

    // Get file size
    off_t size = lseek(fd, 0, SEEK_END);

    // Map the entire file into memory
    char* data = (char*)mmap(
        nullptr,          // let OS choose address
        size,             // length to map
        PROT_READ | PROT_WRITE,  // permissions
        MAP_SHARED,       // changes are written back to file
        fd,               // file descriptor
        0                 // offset in file
    );

    close(fd);  // fd can be closed after mmap — mapping persists

    // Access file contents directly via pointer
    printf("First byte: %c\n", data[0]);

    // Modify the file
    data[0] = 'X';  // this will be written back to the file

    // Unmap when done
    munmap(data, size);
    return 0;
}
```

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 5.3 How mmap Works Internally</h2>

```
Step 1: mmap() call
  - Creates a VMA (Virtual Memory Area) in the process's page table
  - NO physical memory allocated yet!

Step 2: First access to the mapped region
  - Page fault! (page not present in physical memory)
  - Kernel checks: "This is an mmap'd file page"
  - If page is in page cache → map that physical frame directly
  - If not in page cache → read from disk into page cache, then map it
  - Resume execution — transparent to the process

Step 3: Subsequent accesses
  - No page faults (page already mapped)
  - Direct memory access speed (after initial fault)

┌──────────────────────────────────────┐
│ Virtual page (your process)          │
│         │                            │
│         ▼                            │
│  Page Table Entry ──► Physical Frame │  ← same frame as page cache!
│                       (Page Cache)   │
│                          │           │
│                       Disk File      │
└──────────────────────────────────────┘
```

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 5.4 MAP_SHARED vs MAP_PRIVATE</h2>

```
MAP_SHARED:
  - Changes to the memory ARE written back to the file
  - Multiple processes mapping the same file SEE each other's changes
  - Used for: shared memory IPC, memory-mapped databases

MAP_PRIVATE:
  - Uses copy-on-write: initially shares physical pages with file
  - Writes create PRIVATE copies — changes are NOT written to the file
  - Used for: loading shared libraries (.so), private read-modify scenarios
```

```
MAP_SHARED example — two processes:
  Process A maps file.dat with MAP_SHARED
  Process B maps file.dat with MAP_SHARED
  
  Process A writes data[100] = 42
  → Process B immediately sees data[100] == 42
  → File on disk will also reflect the change (eventually, or on msync)

MAP_PRIVATE example:
  Process A maps file.dat with MAP_PRIVATE
  
  A reads data[100] → reads from shared page cache (cheap)
  A writes data[100] = 42 → COW: kernel copies the page, A gets private copy
  → File on disk unchanged
  → Other processes see original data
```

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 5.5 Anonymous mmap — Not Backed by a File</h2>

`mmap` can also allocate memory not backed by any file — this is how `malloc` allocates large blocks:

```cpp
// Anonymous mapping — just raw memory, no file
void* mem = mmap(
    nullptr, 
    1024 * 1024,            // 1 MB
    PROT_READ | PROT_WRITE,
    MAP_PRIVATE | MAP_ANONYMOUS,
    -1,                     // no file descriptor
    0
);

// This is how malloc internally allocates large chunks (> 128KB by default)
// For small allocations, malloc uses brk/sbrk instead
```

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 5.6 <code>msync()</code> — Flushing Changes to Disk</h2>

With `MAP_SHARED`, changes are written to the file — but not necessarily immediately. The kernel may buffer them in the page cache. To force a write:

```cpp
msync(data, size, MS_SYNC);   // blocks until data is written to disk
msync(data, size, MS_ASYNC);  // schedules write, returns immediately
```

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 5.7 mmap vs read/write — When to Use Which</h2>

| | `mmap` | `read`/`write` |
|---|--------|----------------|
| **Best for** | Random access to large files | Sequential reads/writes |
| **Copies** | Zero-copy (page cache mapped directly) | One extra copy (kernel → user) |
| **Syscall overhead** | One `mmap` call, then just memory access | One `read`/`write` per operation |
| **Small files** | Overhead of VMA setup not worth it | Better — simpler, lower setup cost |
| **Large files** | Excellent — virtual memory handles paging | Need to manage buffering yourself |
| **Portability** | POSIX (Linux, macOS, BSD) | Universal |
| **Drawbacks** | SIGBUS on truncation, TLB pressure, hard to handle errors | Extra copy, syscall per operation |

<br>

#### The SIGBUS Trap

If the file is truncated (by another process) while mapped, accessing the now-invalid pages causes `SIGBUS` — not `SIGSEGV`. This is a common gotcha with `mmap`.

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 5.8 Real-World Uses of mmap</h2>

| Use Case | Details |
|----------|---------|
| **Shared libraries (.so)** | `ld.so` uses `mmap(MAP_PRIVATE)` to load `.text` and `.rodata` sections |
| **Memory-mapped databases** | LMDB, SQLite WAL mode use `mmap` for fast reads |
| **Large file processing** | Video editors, log analyzers map files for random access |
| **Shared memory IPC** | `mmap(MAP_SHARED)` on the same file = inter-process communication |
| **malloc large allocations** | glibc `malloc` uses `mmap` for allocations > ~128KB |
| **JIT compilers** | `mmap` with `PROT_EXEC` for executable memory regions |

<br><br>

---

## Interview Questions

1. "What is `mmap`? How is it different from `read()`/`write()`?"
2. "How does mmap achieve zero-copy file access?"
3. "What is the difference between `MAP_SHARED` and `MAP_PRIVATE`?"
4. "What happens when you first access an mmap'd page?"
5. "When would you choose mmap over read/write?"
6. "What is `SIGBUS` in the context of mmap?"
7. "How does `malloc` use mmap internally?"

---

**Next**: [Dynamic Linking →](06-dynamic-linking.md)
