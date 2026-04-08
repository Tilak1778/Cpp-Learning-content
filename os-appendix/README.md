# OS Concepts for Systems C++ Engineers

[← Back to Main Study Plan](../lld-study-plan.md)

> **Purpose**: Supplementary OS theory for platform/systems C++ interview prep.
> These topics don't require build exercises — they're "know and explain" material.
> Study 15-20 min/day alongside the main LLD plan.

---

## Topics

| # | Topic | File |
|---|-------|------|
| 1 | [Virtual Memory](01-virtual-memory.md) | Page tables, TLB, demand paging, page faults |
| 2 | [Process Model](02-process-model.md) | fork, exec, wait, address spaces, process vs thread |
| 3 | [Context Switching & Scheduling](03-context-switching.md) | Kernel/user mode, syscalls, CFS scheduler |
| 4 | [Signals](04-signals.md) | SIGTERM, SIGSEGV, async-signal-safe, signalfd |
| 5 | [Memory-Mapped I/O](05-mmap.md) | mmap for files, page cache, msync |
| 6 | [Dynamic Linking](06-dynamic-linking.md) | dlopen, dlsym, PLT/GOT, shared libraries, ABI |
| 7 | [File System Internals](07-file-system.md) | Inodes, blocks, directory entries, VFS |
| 8 | [Pipes & IPC](08-pipes-ipc.md) | pipe, FIFO, Unix domain sockets, comparison of IPC methods |

---

## Recommended Reading

| Resource | Chapters |
|----------|----------|
| *The Linux Programming Interface* — Michael Kerrisk | Ch 6 (Processes), Ch 24 (Process Creation), Ch 49 (Memory Mapping), Ch 20-22 (Signals) |
| *Operating Systems: Three Easy Pieces* (free online) | Virtualization section (VM, Paging), Concurrency section |
| *Computer Systems: A Programmer's Perspective* (CS:APP) | Ch 9 (Virtual Memory), Ch 7 (Linking) |
