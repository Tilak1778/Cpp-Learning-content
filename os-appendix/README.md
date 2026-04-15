# OS Concepts for Systems C++ Engineers

[← Back to Main Study Plan](../lld-study-plan.md)

> **Purpose**: Supplementary OS theory for platform/systems C++ interview prep.
> These topics don't require build exercises — they're "know and explain" material.
> Study 15-20 min/day alongside the main LLD plan.
>
> **Minix 3 Reference**: Each topic includes a "Minix 3 Deep Dive" section with
> simplified source code from the Minix 3 microkernel OS. Minix 3's clean separation
> of kernel, servers, and drivers makes OS concepts concrete and traceable.
> Source: [github.com/Stichting-MINIX-Research-Foundation/minix](https://github.com/Stichting-MINIX-Research-Foundation/minix)

---

## Minix 3 Architecture Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                      User Processes                             │
│    (your C++ program, shell, utilities)                         │
└──────────┬──────────────────────────────────────────────────────┘
           │ system calls (messages)
┌──────────▼──────────────────────────────────────────────────────┐
│                    System Servers (user-space)                   │
│  ┌─────┐  ┌─────┐  ┌─────┐  ┌─────┐  ┌─────┐  ┌──────┐       │
│  │ PM  │  │ VFS │  │ VM  │  │ RS  │  │ DS  │  │ MFS  │ ...   │
│  │     │  │     │  │     │  │     │  │     │  │      │       │
│  │fork │  │read │  │mmap │  │self │  │name │  │inode │       │
│  │exec │  │write│  │page │  │heal │  │svc  │  │block │       │
│  │wait │  │open │  │fault│  │     │  │     │  │dir   │       │
│  │sigs │  │pipe │  │COW  │  │     │  │     │  │cache │       │
│  └─────┘  └─────┘  └─────┘  └─────┘  └─────┘  └──────┘       │
└──────────┬──────────────────────────────────────────────────────┘
           │ kernel calls (messages)
┌──────────▼──────────────────────────────────────────────────────┐
│              Device Drivers (user-space)                         │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐        │
│  │  Disk    │  │ Network  │  │ Console  │  │  TTY     │ ...    │
│  │  Driver  │  │ Driver   │  │ Driver   │  │ Driver   │        │
│  └──────────┘  └──────────┘  └──────────┘  └──────────┘        │
└──────────┬──────────────────────────────────────────────────────┘
           │ privileged kernel calls
┌──────────▼──────────────────────────────────────────────────────┐
│                    Microkernel (~12,000 lines)                   │
│                                                                 │
│  IPC (send/receive/notify)  •  Scheduling  •  Interrupts       │
│  Context switching  •  Page table management  •  Clock          │
│                                                                 │
│  This is ALL that runs in kernel mode (Ring 0)                  │
└─────────────────────────────────────────────────────────────────┘
```

**Key**: PM = Process Manager, VFS = Virtual File System, VM = Virtual Memory,
RS = Reincarnation Server, DS = Data Store, MFS = Minix File System

---

## Topics

| # | Topic | File | Minix 3 Code |
|---|-------|------|--------------|
| 1 | [Virtual Memory](01-virtual-memory.md) | Page tables, TLB, demand paging, page faults | `servers/vm/` — pagefaults.c, region.c, fork.c |
| 2 | [Process Model](02-process-model.md) | fork, exec, wait, address spaces, process vs thread | `servers/pm/` — forkexit.c, exec.c, mproc.h |
| 3 | [Context Switching & Scheduling](03-context-switching.md) | Kernel/user mode, syscalls, CFS scheduler | `kernel/proc.c` — mini_send, mini_receive, scheduling |
| 4 | [Signals](04-signals.md) | SIGTERM, SIGSEGV, async-signal-safe, signalfd | `servers/pm/signal.c`, `kernel/system/do_sigsend.c` |
| 5 | [Memory-Mapped I/O](05-mmap.md) | mmap for files, page cache, msync | `servers/vm/mmap.c`, mem_anon.c, mem_file.c |
| 6 | [Dynamic Linking](06-dynamic-linking.md) | dlopen, dlsym, PLT/GOT, shared libraries, ABI | `servers/pm/exec.c`, `servers/vfs/exec.c` (ELF loading) |
| 7 | [File System Internals](07-file-system.md) | Inodes, blocks, directory entries, VFS | `servers/vfs/`, `servers/mfs/` — inode.c, cache.c, path.c |
| 8 | [Pipes & IPC](08-pipes-ipc.md) | pipe, FIFO, Unix domain sockets, message passing | `kernel/proc.c` — THE IPC core of the microkernel |

---

## Recommended Reading

| Resource | Chapters |
|----------|----------|
| *Operating Systems: Design and Implementation* (3rd ed.) — Andrew S. Tanenbaum, Albert S. Woodhull | The definitive Minix 3 textbook. Ch 1-2 (Architecture), Ch 3 (IPC & Process Mgmt), Ch 4 (Memory Mgmt), Ch 5 (File Systems) |
| *The Linux Programming Interface* — Michael Kerrisk | Ch 6 (Processes), Ch 24 (Process Creation), Ch 49 (Memory Mapping), Ch 20-22 (Signals) |
| *Operating Systems: Three Easy Pieces* (free online) | Virtualization section (VM, Paging), Concurrency section |
| *Computer Systems: A Programmer's Perspective* (CS:APP) | Ch 9 (Virtual Memory), Ch 7 (Linking) |
| Minix 3 Source Code | [github.com/Stichting-MINIX-Research-Foundation/minix](https://github.com/Stichting-MINIX-Research-Foundation/minix) — browse `minix/kernel/`, `minix/servers/`, `minix/drivers/` |
| Minix 3 Wiki | [wiki.minix3.org](https://wiki.minix3.org) — architecture docs, build instructions |

---

## Why Minix 3 for Learning?

| | Linux Kernel | Minix 3 |
|---|-------------|---------|
| Kernel lines of code | ~30 million | ~12,000 (microkernel only) |
| Architecture | Monolithic (everything in Ring 0) | Microkernel (only IPC, scheduling, interrupts in Ring 0) |
| Crash impact | Any kernel bug → system panic | Driver/server crash → automatic restart |
| Learning curve | Steep (complex, production-optimized) | Gentle (textbook-quality, designed for teaching) |
| Self-healing | No | Yes (Reincarnation Server restarts failed components) |
| Performance | Fastest (minimal overhead) | Slower (~5-10% overhead from IPC) |
| Best for | Production systems | Understanding OS fundamentals |
