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

## Minix 3 Deep Dive — mmap in the VM Server

Minix 3's mmap implementation lives in the VM server (`servers/vm/mmap.c`), making it one of the most readable mmap implementations available for study.

<br>

<h3 style="color: #E67E22;">Source Files</h3>

```
minix/servers/vm/
├── mmap.c         ← mmap() and munmap() syscall handlers
├── region.c       ← virtual memory region management
├── mem_anon.c     ← anonymous memory backend (MAP_ANONYMOUS)
├── mem_file.c     ← file-backed memory backend (file mmap)
├── mem_shared.c   ← shared memory backend (MAP_SHARED)
├── pagefaults.c   ← page fault handler (demand paging for mmap'd regions)
└── region.h       ← struct vir_region definition
```

<br>

<h3 style="color: #E67E22;">How mmap() Is Processed</h3>

When a user calls `mmap()`, the libc sends a message to the VM server:

```
User calls:  mmap(NULL, 4096, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_ANON, -1, 0)
     │
     ▼
libc packs message → sends to VM server
     │
     ▼
VM server: do_mmap() in servers/vm/mmap.c
```

```c
/* servers/vm/mmap.c (simplified) */
int do_mmap(message *msg) {
    struct vmproc *vmp = find_proc(msg->m_source);
    vir_bytes addr  = msg->VMM_ADDR;    /* requested address (or 0 for "any") */
    size_t length   = msg->VMM_LEN;
    int prot        = msg->VMM_PROT;    /* PROT_READ, PROT_WRITE, PROT_EXEC */
    int flags       = msg->VMM_FLAGS;   /* MAP_SHARED, MAP_PRIVATE, MAP_ANON */
    int fd          = msg->VMM_FD;
    off_t offset    = msg->VMM_OFFSET;

    /* Round length up to page boundary */
    length = roundup(length, PAGE_SIZE);

    /* Choose memory type backend based on flags */
    struct mem_type *memtype;
    if (flags & MAP_ANONYMOUS) {
        memtype = &mem_type_anon;       /* no file backing */
    } else {
        memtype = &mem_type_file;       /* file-backed */
    }

    /* Set region flags */
    u32_t vrflags = 0;
    if (prot & PROT_WRITE) vrflags |= VR_WRITABLE;
    if (flags & MAP_SHARED) vrflags |= VR_SHARED;

    /* Find free virtual address space if addr == 0 */
    if (addr == 0) {
        addr = find_free_region(vmp, length);
    }

    /* Create a new virtual memory region */
    struct vir_region *vr = map_page_region(
        vmp, addr, length, vrflags, memtype
    );

    if (memtype == &mem_type_file) {
        /* Set up file backing: tell VFS which file and offset */
        vr->file_inode = get_inode_for_fd(vmp, fd);
        vr->file_offset = offset;
    }

    /* No physical memory allocated yet — demand paging will handle it */

    /* Return the mapped address to the caller */
    msg->VMM_RETADDR = addr;
    return OK;
}
```

The critical insight: `mmap()` only creates a **region descriptor** — no physical pages are allocated. The first access triggers a page fault, and the VM server's `pagefaults.c` handles it by calling the appropriate backend.

<br>

<h3 style="color: #E67E22;">Page Fault for Anonymous mmap</h3>

```c
/* servers/vm/mem_anon.c (simplified) */
int anon_pagefault(struct vmproc *vmp, struct vir_region *region,
                   struct phys_region *ph, int write) {
    /* Allocate a fresh physical page */
    phys_bytes new_page = alloc_mem(1);  /* 1 page = 4KB */

    /* Zero it out (anonymous memory is always zeroed) */
    phys_zero(new_page, PAGE_SIZE);

    /* Create a physical region entry and link it to the virtual region */
    ph = pb_insert(region, offset, new_page);

    /* Install the mapping in the page table */
    pt_writemap(&vmp->vm_pt, fault_addr, new_page,
                PAGE_SIZE, region->flags & VR_WRITABLE ? PT_WRITABLE : 0);

    return OK;
}
```

<br>

<h3 style="color: #E67E22;">Page Fault for File-Backed mmap</h3>

```c
/* servers/vm/mem_file.c (simplified) */
int file_pagefault(struct vmproc *vmp, struct vir_region *region,
                   struct phys_region *ph, int write) {
    /* Check if this page is already in the page cache */
    struct cached_page *cp = find_cached_page(region->file_inode,
                                               region->file_offset + offset);

    if (cp) {
        /* Cache hit — just map the existing physical page */
        ph = pb_insert(region, offset, cp->phys);
        cp->refcount++;
    } else {
        /* Cache miss — allocate a page and read from disk */
        phys_bytes new_page = alloc_mem(1);

        /* Ask VFS to read the file data into this page */
        req_readwrite(region->file_inode, region->file_offset + offset,
                      vmp->vm_endpoint, new_page, PAGE_SIZE);

        /* Add to page cache for future accesses */
        cache_insert(region->file_inode, region->file_offset + offset,
                     new_page);

        ph = pb_insert(region, offset, new_page);
    }

    /* MAP_PRIVATE + write → need copy-on-write */
    if (write && !(region->flags & VR_SHARED) && ph->refcount > 1) {
        phys_bytes cow_page = alloc_mem(1);
        phys_copy(ph->phys, cow_page, PAGE_SIZE);
        ph->phys = cow_page;
        ph->refcount = 1;
    }

    pt_writemap(&vmp->vm_pt, fault_addr, ph->phys,
                PAGE_SIZE, write ? PT_WRITABLE : 0);

    return OK;
}
```

<br>

<h3 style="color: #E67E22;">munmap() — Cleaning Up</h3>

```c
/* servers/vm/mmap.c (simplified) */
int do_munmap(message *msg) {
    struct vmproc *vmp = find_proc(msg->m_source);
    vir_bytes addr = msg->VMM_ADDR;
    size_t length  = msg->VMM_LEN;

    struct vir_region *vr = find_region(vmp, addr);

    /* For each physical page in the region */
    for (each phys_region in vr, covering addr to addr+length) {
        ph->refcount--;
        if (ph->refcount == 0) {
            /* Last reference — free the physical page */
            free_mem(ph->phys, 1);
        }
        /* Remove page table entry */
        pt_writemap(&vmp->vm_pt, page_addr, 0, PAGE_SIZE, 0);
    }

    /* Remove the region from the process's region list */
    region_remove(vmp, vr, addr, length);

    return OK;
}
```

<br>

<h3 style="color: #E67E22;">brk/sbrk — Heap Growth in Minix 3</h3>

The heap is managed through `brk()`, which is simpler than mmap but uses the same VM infrastructure:

```c
/* servers/vm/break.c (simplified) */
int do_brk(message *msg) {
    struct vmproc *vmp = find_proc(msg->m_source);
    vir_bytes new_brk = msg->VMB_ADDR;

    struct vir_region *heap_region = find_heap_region(vmp);

    if (new_brk > current_brk) {
        /* Growing the heap — extend the region */
        region_extend(heap_region, new_brk - current_brk);
    } else if (new_brk < current_brk) {
        /* Shrinking — free pages and shrink region */
        region_shrink(heap_region, current_brk - new_brk);
    }

    return OK;
}
```

This is what `malloc` calls internally for small allocations (below the mmap threshold). For large allocations, glibc's malloc calls `mmap()` directly.

<br>

<h3 style="color: #E67E22;">Microkernel Advantage: mmap Fault Isolation</h3>

```
Linux: Bug in mmap handler (mm/mmap.c) → kernel panic → system dead

Minix 3: Bug in VM server's mmap.c → VM server crashes
  → Reincarnation Server (RS) detects the crash
  → RS restarts the VM server
  → Running processes may be affected, but kernel is fine
  → System can potentially recover

This is why Minix 3 is used in high-reliability scenarios
(e.g., it was considered for pacemaker firmware).
```

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
8. "Explain the page fault handling path for a file-backed mmap region."
9. "What is the memory type backend pattern used by Minix 3's VM server?"

---

**Next**: [Dynamic Linking →](06-dynamic-linking.md)
