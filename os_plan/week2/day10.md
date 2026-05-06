# Day 10 — Page tables in detail

> **Week 2 · Memory**
> Reading: OSTEP Chapters 20 (Smaller Tables); LKD memory chapters

## Why this matters

Yesterday: paging at the conceptual level. Today: the actual data structures. Multi-level page tables are how we map huge virtual address spaces with manageable memory overhead. This is dense material but pays off — every "what does the MMU actually do" interview question gets resolved here.

## 10.1 Why multi-level page tables?

A flat page table mapping every virtual page would need 2^36 entries (with 48-bit virtual addresses and 4 KB pages) × 8 bytes = 512 GB **per process**. Impossible.

The insight: most of the address space is unmapped. A typical process uses maybe a few hundred MB of actual mappings out of 256 TB possible. A multi-level tree only needs to populate branches that have content; sparse regions cost nothing.

## 10.2 x86-64 4-level paging

x86-64 (the common case) uses 4 levels of page tables. A 48-bit virtual address splits into:

```
  47          39 38         30 29         21 20         12 11           0
┌──────────────┬──────────────┬──────────────┬──────────────┬─────────────┐
│  PML4 idx    │   PDPT idx   │    PD idx    │    PT idx    │   Offset    │
│  (9 bits)    │  (9 bits)    │  (9 bits)    │  (9 bits)    │  (12 bits)  │
└──────────────┴──────────────┴──────────────┴──────────────┴─────────────┘
```

Each level is a 4 KB table containing 512 entries (each 8 bytes). The translation walk:

```mermaid
flowchart LR
    CR3["CR3 (page table base)"] --> PML4
    PML4["PML4 table<br/>(top level)"] -->|"vaddr[47:39]"| PDPT
    PDPT["PDPT table"] -->|"vaddr[38:30]"| PD
    PD["PD table"] -->|"vaddr[29:21]"| PT
    PT["PT table"] -->|"vaddr[20:12]"| Frame
    Frame["Physical frame"] -->|"+ vaddr[11:0]"| PA["Physical address"]

    style CR3 fill:#1F3864,color:#fff
    style PA fill:#2E75B6,color:#fff
```

The CPU register **CR3** holds the physical address of the top-level (PML4) table for the current process. On context switch (between processes with different `mm_struct`s), the kernel updates CR3.

### Naming in Linux source

The Linux kernel uses architecture-neutral names:

- **PGD** (Page Global Directory) ≈ PML4 on x86-64
- **PUD** (Page Upper Directory) ≈ PDPT
- **PMD** (Page Middle Directory) ≈ PD
- **PTE** (Page Table Entry) — leaf level

Plus **P4D** for the 5-level case. So Linux source talks about pgd → p4d → pud → pmd → pte.

## 10.3 What's in a PTE?

A PTE (or PMD/PUD/PML4 entry — same format) is 64 bits:

```
 63  62              52 51                                     12 11       0
┌────┬─────────────────┬───────────────────────────────────────┬──────────┐
│ XD │  Reserved/SW    │      Physical frame number (40 bits)  │  Flags   │
│NX  │                 │                                       │          │
└────┴─────────────────┴───────────────────────────────────────┴──────────┘
```

The flag bits (offset within 4 KB irrelevant, those bits hold flags):

| Bit | Name | Meaning |
|-----|------|---------|
| 0 | P (Present) | Page is in physical memory |
| 1 | R/W | Writable |
| 2 | U/S | User-accessible |
| 3 | PWT | Page write-through |
| 4 | PCD | Page cache-disabled |
| 5 | A (Accessed) | Set by hardware on read or write |
| 6 | D (Dirty) | Set by hardware on write |
| 7 | PS | Page size (0 = 4 KB; 1 = 2 MB at PMD level, 1 GB at PUD) |
| 8 | G (Global) | Don't flush on CR3 change |
| 9–11 | SW | Software-defined (kernel uses for swap, etc.) |
| 12–51 | PFN | Physical frame number |
| 63 | XD | Execute disable (NX bit) |

If the **Present bit is 0**, the rest of the entry can be repurposed by the OS — typically to encode a swap entry (which swap area, what offset).

## 10.4 Walking through an example

Suppose process X has a mapping for virtual address 0x7fff_aabb_c123. Let's translate:

```
binary: 01111111 11111111 10101010 10111011 11000001 0010 0011

PML4 idx (bits 47..39) = 0b011111111 = 255
PDPT idx (bits 38..30) = 0b111111110 = 510
PD   idx (bits 29..21) = 0b101010110 = 342
PT   idx (bits 20..12) = 0b111110000 = 496
Offset (bits 11..0)    = 0b000100100011 = 0x123
```

Walk:

1. CPU reads CR3 → physical address of PML4 (e.g., 0x4000).
2. Read PML4[255] → entry contains physical address of PDPT (e.g., 0x5000) and present=1.
3. Read PDPT[510] → entry contains PD address (e.g., 0x6000), present=1.
4. Read PD[342] → entry contains PT address (e.g., 0x7000), present=1, PS=0.
5. Read PT[496] → entry contains physical frame (e.g., 0x12345 << 12 = 0x12345000), present=1, U/S=1, R/W=1.
6. Combine frame 0x12345000 + offset 0x123 = physical address 0x12345123.

That's the translation. Five memory accesses (CR3 + 4 table reads + offset = data). Without a TLB, every single user-mode memory access would do this.

### What if PS=1?

If a level has its PS (Page Size) bit set, that level is a leaf for a huge page:

- PD entry with PS=1: 2 MB page. Bits below are the page offset (21 bits).
- PUD entry with PS=1: 1 GB page. Offset is 30 bits.

This is how huge pages work: skip a level, get a bigger frame.

## 10.5 5-level paging

x86-64 originally used 4-level (48-bit virtual). Modern CPUs (Intel Ice Lake+, some others) support **5-level paging** for 57-bit virtual addresses (128 PB). Same idea, one more level (P4D / PML5) at the top. Linux supports it; mostly relevant for huge servers with terabytes of RAM. Userspace defaults to 4-level for compatibility unless a process opts in via mmap with a high hint.

## 10.6 Page table memory cost

For a fully-mapped 256 TB process: 4 levels of 512-entry tables, 8 bytes each, but total leaf-level entries are 2^36 = 64 G. At 8 bytes each: 512 GB just for leaf PTEs. So even the multi-level structure can't help if you map everything.

Real processes map far less. A typical 100 MB-resident process has maybe ~100K leaf PTEs = 800 KB of leaf tables, plus a few KB of higher levels. Total page-table overhead: typically a few MB.

You can see this:

```bash
cat /proc/$$/status | grep -E 'VmPTE|VmRSS|VmSize'
```

`VmPTE` is the page-table size for that process.

## 10.7 Sharing page tables

Threads share the `mm_struct`, which means they share page tables — there's exactly one set of tables for all threads of a process. CR3 is the same for all of them.

For independent processes, page tables are private. **However**, the kernel mappings (top half of address space) are shared: every process's PML4 has the same upper-half entries, pointing at the same kernel PDPTs. So when a syscall enters the kernel, the same kernel data structures are visible regardless of which process called.

KPTI (post-Meltdown) breaks this slightly: user-mode CR3 has minimal kernel mappings (just a trampoline), kernel-mode CR3 has the full kernel.

## 10.8 What happens on context switch

When the kernel switches from process A to process B:

1. Save A's CPU state.
2. If B has a different `mm_struct`: load B's pgd into CR3.
3. (PCID-aware): tag the new CR3 with B's PCID; old PCID's entries stay in TLB.
4. Restore B's state.
5. Resume.

If A and B are threads of the same process, CR3 doesn't change. Only registers swap. Much faster.

### Lazy TLB

Optimization: kernel threads (no `mm_struct`) don't need to switch CR3 at all. They "borrow" whatever CR3 was last loaded. Linux's `lazy_mm` mechanism tracks this and avoids unnecessary CR3 writes. Same idea for switching to and from idle.

## Hands-on (30 minutes)

1. Look at your shell's page table size:
   ```bash
   cat /proc/$$/status | grep VmPTE
   ```
   Note the units (KB).

2. Compare across a small process and a large one:
   ```bash
   cat /proc/$(pgrep -n bash)/status | grep -E 'VmPTE|VmRSS'
   cat /proc/$(pgrep -n firefox)/status | grep -E 'VmPTE|VmRSS' 2>/dev/null
   ```
   Bigger RSS → bigger VmPTE.

3. See accessed/dirty bits in action via /proc/$$/pagemap. This is advanced but doable. Read `Documentation/admin-guide/mm/pagemap.rst`.

4. Inspect the kernel's view of your mappings:
   ```bash
   sudo cat /sys/kernel/debug/page_tables/init  # if available
   ```
   This dumps the kernel's page tables (debugfs needs to be mounted).

5. Trigger a TLB flush by remapping memory and observe the latency cost. (Hard to do directly; usually visible in `perf` traces of `munmap`-heavy workloads.)

## Interview questions

### Q1. Walk me through a virtual-to-physical address translation on x86-64.

**Answer:** The CPU emits a 48-bit virtual address, split into five fields by bit position:

- Bits 47–39: PML4 index
- Bits 38–30: PDPT index
- Bits 29–21: PD index
- Bits 20–12: PT index
- Bits 11–0: page offset

Translation walks four levels of tables:

1. CR3 holds the physical address of the PML4 (top-level) table for the current process. Read PML4[idx]; the entry holds the physical address of the next level (PDPT).
2. Use the PDPT index to find the PDPT entry; that points to a PD.
3. Use the PD index for the PD entry; that points to a PT (or, if PS=1, points directly to a 2 MB huge page).
4. Use the PT index for the PTE; that contains the physical frame number.
5. Concatenate frame number with the 12-bit offset to get the physical address.

Each level's entry is 8 bytes; each table is 4 KB (512 entries). The MMU caches the result in the TLB. Subsequent accesses to the same page hit the TLB and skip the walk.

If any level's entry has Present=0, the access faults — the kernel handles via the page fault handler (Day 11). Permission bits at each level constrain access; the most restrictive permission across the walk wins.

### Q2. Why are page tables organized as a tree instead of a flat array?

**Answer:** With 48-bit virtual addresses and 4 KB pages, a flat table would need 2^36 entries × 8 bytes = 512 GB per process. A tree only allocates the levels that have content; sparsely-populated address spaces (every typical process) consume far less.

Concretely: most of a process's 256 TB virtual space is unmapped. A flat table would allocate 512 GB of zeros. A tree only allocates the branches you actually use. A typical 100 MB process uses a few MB of page tables.

The tradeoff: each translation requires multiple memory loads (one per level). Without a TLB, this would be devastating. The TLB caches recent translations, making the common case ~free.

You can think of it like a B-tree for sparse mappings — paying a small structural cost to avoid massive empty regions.

### Q3. What does the kernel do on a context switch with respect to page tables?

**Answer:** The kernel updates CR3 to point to the new process's PGD (PML4 on x86-64). This single instruction switches the entire address space — every subsequent virtual-address access uses the new tables.

Side effects:

- TLB entries from the old process are stale. Without help, the CPU must flush. This is expensive: every memory access for a while takes a TLB miss.
- Modern CPUs support PCID (Process-Context IDentifier): each TLB entry is tagged with the CR3's PCID. Switching to a process whose entries are still cached uses them. When you switch back, no full flush.
- Kernel mappings are usually marked Global (bit 8); they aren't flushed on CR3 change. Post-Meltdown KPTI complicates this: user-mode CR3 has minimal kernel mappings.

If both tasks are threads of the same process (same `mm_struct`), CR3 doesn't change — only registers. Much cheaper.

If switching to a kernel thread (no `mm`), Linux uses a lazy TLB optimization: don't bother changing CR3, since kernel threads don't access user mappings. Borrow whatever CR3 the previous task had.

### Q4. What are the bits in a PTE? What does each do?

**Answer:** A 64-bit PTE on x86-64 contains:

- **Present (P)**: page is in physical memory. If 0, accessing the page faults; rest of the entry is software-defined (often a swap entry).
- **Read/Write (R/W)**: writes allowed. If 0, writes fault.
- **User/Supervisor (U/S)**: user-mode access allowed. If 0, only kernel can access.
- **Accessed (A)**: set by hardware on first read or write. The kernel uses this for LRU approximation.
- **Dirty (D)**: set by hardware on first write. Tells the kernel the page has been modified — must be written to disk before reclaim if it's file-backed.
- **PWT, PCD**: cache attributes (write-through, cache disable).
- **PS (Page Size)**: at intermediate levels, this turns the entry into a leaf for a huge page (2 MB at PD level, 1 GB at PUD level).
- **Global (G)**: don't flush on CR3 change. Used for kernel pages so they survive context switches.
- **Software bits** (9–11, plus some 52–62): the OS uses these for its own purposes — flagging COW, special states, etc.
- **Frame number (12–51)**: 40 bits, locating the physical 4 KB frame.
- **NX/XD (63)**: Non-eXecutable / eXecute Disable. Hardware refuses to fetch instructions from this page if set. Used to mark stack and heap as data-only, defending against many exploits.

When the kernel changes any of these (e.g., enabling write on a COW page after copying), it must invalidate the relevant TLB entry — `invlpg` instruction, or full flush, or shootdown across CPUs.

## Self-test

1. On x86-64 with 4-level paging, how many memory accesses does a single virtual-to-physical translation require if the TLB misses entirely?
2. The Accessed bit gets set by hardware. What does the kernel use it for?
3. What's the difference between a 4 KB page table walk and a 2 MB huge page lookup?
4. The PFN field in a PTE is 40 bits. With 4 KB pages, what's the maximum physical address the system can reference?
5. KPTI reduces a security risk but adds CPU overhead. What's the risk and what's the overhead?
