# OS Appendix 1: Virtual Memory

[← Back to OS Appendix](README.md)

---
---

# VIRTUAL MEMORY — THE FULL PICTURE

---

<br>

<h2 style="color: #2980B9;">📘 1.1 Why Virtual Memory Exists</h2>

Without virtual memory, every process would directly access physical RAM:

```
Process A: writes to address 0x1000
Process B: also writes to address 0x1000
→ They overwrite each other! No isolation.
```

Virtual memory gives each process its own **private address space** — an illusion that it has the entire memory to itself:

```
Process A sees:   0x0000 ─────── 0xFFFF...F   (its own virtual addresses)
Process B sees:   0x0000 ─────── 0xFFFF...F   (its own virtual addresses)

Both use address 0x1000, but the hardware maps them to DIFFERENT physical locations:
  A's 0x1000 → physical 0x50000
  B's 0x1000 → physical 0x80000
```

**Three key benefits:**
1. **Isolation** — processes can't corrupt each other's memory
2. **Simplicity** — every process thinks it starts at address 0
3. **Overcommit** — you can allocate more virtual memory than physical RAM exists (the OS pages to disk)

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 1.2 Pages — The Unit of Memory</h2>

Virtual memory works in fixed-size chunks called **pages** (typically **4 KB** on x86/ARM):

```
Virtual Address Space (per process):
┌──────────────┐  0xFFFF...F
│    Stack     │  ↓ grows down
│     ...      │
│              │
│    Heap      │  ↑ grows up
│     ...      │
│  .data/.bss  │  global/static variables
│    .text     │  executable code
└──────────────┘  0x0000...0

Each of these regions is divided into 4KB pages:
  Page 0: 0x0000 – 0x0FFF
  Page 1: 0x1000 – 0x1FFF
  Page 2: 0x2000 – 0x2FFF
  ...
```

Physical RAM is also divided into 4KB chunks called **frames**:

```
Physical RAM:
┌────────┬────────┬────────┬────────┬────────┬─────┐
│Frame 0 │Frame 1 │Frame 2 │Frame 3 │Frame 4 │ ... │
└────────┴────────┴────────┴────────┴────────┴─────┘
```

The OS maps virtual pages to physical frames — they don't need to be contiguous:

```
Process A virtual pages → Physical frames:
  Page 0 → Frame 7
  Page 1 → Frame 2
  Page 2 → Frame 15
  Page 3 → (not mapped — not allocated yet)
```

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 1.3 Page Table — The Translation Map</h2>

Each process has a **page table** — a data structure that maps virtual page numbers to physical frame numbers:

```
Process A's Page Table:
┌───────────────┬────────────────┬───────────┬──────────┐
│ Virtual Page  │ Physical Frame │ Present?  │ Writable │
├───────────────┼────────────────┼───────────┼──────────┤
│     0         │     7          │    Yes    │   Yes    │
│     1         │     2          │    Yes    │   Yes    │
│     2         │    15          │    Yes    │   No     │  ← read-only (code page)
│     3         │     —          │    No     │   —      │  ← not in RAM
│     4         │    42          │    Yes    │   Yes    │
│    ...        │   ...          │   ...     │  ...     │
└───────────────┴────────────────┴───────────┴──────────┘
```

When the CPU accesses a virtual address, the hardware (**MMU** — Memory Management Unit) does:

```
Virtual address: 0x00001A3C
                 ├──────┤├──┤
                 Page #1  Offset 0xA3C (within the page)

1. Look up Page 1 in page table → Frame 2
2. Physical address = Frame 2 base + Offset = 0x2000 + 0xA3C = 0x2A3C
3. Access physical memory at 0x2A3C
```

This happens on **every single memory access** — the hardware does it in nanoseconds.

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 1.4 Multi-Level Page Tables</h2>

A single flat page table for a 64-bit address space would be enormous (terabytes). Instead, modern CPUs use **multi-level** page tables:

```
x86-64 uses 4 levels (with 48-bit virtual addresses):

Virtual Address (48 bits):
┌─────────┬─────────┬─────────┬─────────┬──────────────┐
│ PML4    │  PDPT   │   PD    │   PT    │   Offset     │
│ (9 bits)│ (9 bits)│ (9 bits)│ (9 bits)│  (12 bits)   │
└─────────┴─────────┴─────────┴─────────┴──────────────┘
  Level 4    Level 3   Level 2   Level 1    Within page

Each level is a table with 512 entries (2^9).
Each entry points to the next level's table (or a physical frame at level 1).
```

```
Translation walk:
  CR3 register → PML4 table
                    ↓ (index from bits 47-39)
                  PDPT table
                    ↓ (index from bits 38-30)
                  PD table
                    ↓ (index from bits 29-21)
                  PT table
                    ↓ (index from bits 20-12)
                  Physical Frame + Offset (bits 11-0)
```

Only the parts of the page table that are actually used get allocated. If a process only uses a small part of the address space, most intermediate tables are never created.

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 1.5 TLB — Translation Lookaside Buffer</h2>

Walking 4 levels of page tables on every memory access would be slow. The CPU caches recent translations in a small, fast hardware cache called the **TLB**:

```
CPU wants to access virtual address 0x1A3C:

Step 1: Check TLB (very fast — 1 cycle)
  ┌─────────────────────────────────────────┐
  │ TLB (typically 64-1024 entries)         │
  │ Virtual Page → Physical Frame           │
  │ Page 1 → Frame 2            ← HIT!     │
  └─────────────────────────────────────────┘
  → Physical address = 0x2A3C, done.

If TLB MISS:
  Step 2: Walk the 4-level page table (~100 cycles)
  Step 3: Store result in TLB for next time
  Step 4: Access physical memory
```

**TLB hit rate** is typically 99%+ in well-behaved programs. A TLB miss is expensive (~100 cycles vs 1 cycle for a hit).

<br>

#### What causes TLB thrashing?

- Randomly accessing huge amounts of memory (larger than TLB can cover)
- Context switches flush the TLB (new process = different page table)
- Large number of small allocations scattered across many pages

**Interview tip**: "Why is iterating a contiguous array faster than traversing a linked list?" — Part of the answer is TLB: arrays keep addresses in the same pages, linked list nodes can be scattered across many pages, causing TLB misses.

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 1.6 Page Faults — When the Page Isn't There</h2>

If the "Present" bit in the page table entry is **No**, the hardware triggers a **page fault** — an exception that transfers control to the OS kernel.

There are three types:

<br>

#### Minor (soft) page fault — page is in RAM but not mapped yet

```
Process calls malloc(4096) → allocator marks a virtual page
Process writes to that page for the FIRST time
→ Page fault!
→ OS allocates a physical frame, zeros it, maps it in the page table
→ Resume execution — transparent to the process
```

This is called **demand paging** — the OS doesn't allocate physical memory until you actually use it. This is why `malloc(1GB)` succeeds even with 1MB of free RAM — virtual pages are allocated, physical frames are not.

<br>

#### Major (hard) page fault — page was swapped to disk

```
OS ran low on RAM, swapped this page to disk (swap space)
Process accesses the page
→ Page fault!
→ OS reads the page from disk back into RAM (SLOW — milliseconds)
→ Update page table, resume execution
```

Major page faults are very expensive (disk I/O). Too many = "thrashing" = system grinds to a halt.

<br>

#### Invalid page fault — illegal access

```
Process accesses unmapped memory (e.g., dereferencing nullptr, stack overflow)
→ Page fault!
→ OS checks: this address is not part of any valid mapping
→ OS sends SIGSEGV (segmentation fault) → process crashes
```

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 1.7 Demand Paging & Overcommit</h2>

**Demand paging** = OS doesn't allocate physical memory until the process writes to a page.

```cpp
char* buf = (char*)malloc(1024 * 1024 * 1024);  // 1 GB virtual allocation
// Physical RAM used: nearly 0 — just page table entries

buf[0] = 'A';      // page fault → OS allocates ONE physical frame (4KB)
buf[4096] = 'B';   // page fault → another 4KB frame
// Physical RAM used: 8 KB (for 1 GB virtual)
```

**Overcommit** = Linux (by default) allows you to allocate more virtual memory than physical RAM + swap combined. The assumption: not all processes will use all their memory simultaneously.

The danger: if everyone actually uses their memory, the **OOM (Out-Of-Memory) killer** activates and kills processes to free RAM.

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 1.8 Copy-on-Write (OS Level)</h2>

You learned COW for strings on Day 6. The OS uses the exact same technique for process memory during `fork()`:

```
Parent process:
  Virtual Page 5 → Physical Frame 20 (writable)

After fork():
  Parent Page 5 → Physical Frame 20 (read-only!)
  Child  Page 5 → Physical Frame 20 (read-only!)  ← SAME frame, shared

Both page table entries marked READ-ONLY.

Child writes to Page 5:
→ Page fault! (write to read-only page)
→ OS allocates Frame 30, copies Frame 20's contents
→ Child  Page 5 → Physical Frame 30 (writable)
→ Parent Page 5 → Physical Frame 20 (writable again)
```

This makes `fork()` very fast — no memory is copied until one process writes. If the child immediately calls `exec()` (common pattern), most pages are never copied at all.

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 1.9 The Process Memory Layout</h2>

```
High addresses    ┌──────────────────────────┐  0x7FFF...
                  │        Stack             │  ← local variables, return addresses
                  │        ↓ grows down      │     set via ulimit (typically 8MB)
                  ├──────────────────────────┤
                  │                          │
                  │     (unmapped gap)       │  ← guard pages, ASLR randomization
                  │                          │
                  ├──────────────────────────┤
                  │     Memory-mapped        │  ← mmap, shared libraries, large malloc
                  │     region               │
                  ├──────────────────────────┤
                  │        ↑ grows up        │
                  │        Heap              │  ← malloc/new allocations
                  │     (brk/sbrk)           │
                  ├──────────────────────────┤
                  │     .bss                 │  ← uninitialized globals (zero-filled)
                  ├──────────────────────────┤
                  │     .data                │  ← initialized globals
                  ├──────────────────────────┤
                  │     .rodata              │  ← string literals, constants (read-only)
                  ├──────────────────────────┤
                  │     .text                │  ← executable code (read-only + execute)
Low addresses     └──────────────────────────┘  0x0000...
```

You can see this for any process: `cat /proc/<pid>/maps` on Linux or `vmmap <pid>` on macOS.

<br><br>

---

## Interview Questions

1. "What is virtual memory? Why does it exist?"
2. "What happens when a process accesses memory for the first time after `malloc`?"
3. "Explain the page table and how virtual-to-physical address translation works."
4. "What is a TLB? What causes TLB thrashing?"
5. "What are the three types of page faults?"
6. "How does `fork()` use copy-on-write?"
7. "Why can `malloc(1GB)` succeed even with 100MB of free RAM?"
8. "Draw the memory layout of a process."

---

**Next**: [Process Model →](02-process-model.md)
