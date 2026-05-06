# 30-Day OS & Linux Deep Study Plan

A focused one-month plan for a systems software engineer (5+ years experience) who wants **deep understanding** of operating systems and Linux internals — and is also preparing for **technical interviews**.

## Goals

By the end of 30 days you will be able to:

- Explain virtual memory, paging, and the TLB from hardware up.
- Describe what `fork`, `exec`, `mmap`, `epoll`, and `fsync` actually do, in detail.
- Reason about concurrency primitives (mutex vs. spinlock vs. RCU vs. lock-free) and when each is appropriate.
- Walk through how a packet, a syscall, or a page fault flows through the Linux kernel.
- Use modern Linux observability tools (`strace`, `perf`, `ftrace`, `bpftrace`) to investigate real systems.
- Answer the OS questions that get asked in systems-software interviews — clearly and from first principles.

## Structure

Roughly **2 hours per day**, ~60 hours total.

| Week | Theme | Days |
|------|-------|------|
| 1 | Processes, threads, scheduling | 1–7 |
| 2 | Memory — virtual memory, paging, allocation | 8–14 |
| 3 | Concurrency, synchronization, IPC | 15–21 |
| 4 | I/O, filesystems, networking, synthesis | 22–30 |

Each day has its own markdown file with:

- **Why this matters** — the concept's real-world relevance
- **Reading list** — pages from primary sources (OSTEP, TLPI, LKD)
- **Core content** — teaching prose with diagrams and code examples
- **Hands-on** — a 30-minute practical task
- **Interview questions** — typical questions on the day's topic, with full answers
- **Self-test** — 5 questions to check understanding

## Primary sources

The plan references three books, all worth owning or accessing:

1. **OSTEP** — *Operating Systems: Three Easy Pieces* by Arpaci-Dusseau. **Free** at [pages.cs.wisc.edu/~remzi/OSTEP](https://pages.cs.wisc.edu/~remzi/OSTEP/). The clearest OS textbook in print.
2. **TLPI** — *The Linux Programming Interface* by Michael Kerrisk. The definitive Linux syscall reference.
3. **LKD** — *Linux Kernel Development*, 3rd ed. by Robert Love. Slim, accessible kernel internals book.

Optional but recommended:

- **Brendan Gregg, *Systems Performance, 2nd ed.*** — for performance/observability.
- **LWN.net** — weekly long-form kernel-development articles.

## How to use this plan

1. Read the day's markdown file in full.
2. Do the assigned reading from the primary sources.
3. Complete the hands-on task — actually run the commands, don't just read about them.
4. Answer the interview questions out loud or in writing **before** reading the model answers.
5. Take the self-test.

The interview questions are not a quiz — they are a rehearsal. Practice articulating answers as if you were explaining to an interviewer or peer. Clarity under spoken pressure is half the battle in interviews.

## Daily index

### Week 1 — Processes, threads, scheduling
- [Day 1 — What is an operating system?](week1/day01.md)
- [Day 2 — Processes: the abstraction](week1/day02.md)
- [Day 3 — fork, exec, wait](week1/day03.md)
- [Day 4 — Threads and the user/kernel split](week1/day04.md)
- [Day 5 — CPU scheduling fundamentals](week1/day05.md)
- [Day 6 — Linux scheduling: CFS and friends](week1/day06.md)
- [Day 7 — Week 1 review and interview drill](week1/day07.md)

### Week 2 — Memory
- [Day 8 — Address spaces and virtual memory](week2/day08.md)
- [Day 9 — Paging and the TLB](week2/day09.md)
- [Day 10 — Page tables in detail](week2/day10.md)
- [Day 11 — Page faults and demand paging](week2/day11.md)
- [Day 12 — mmap and shared memory](week2/day12.md)
- [Day 13 — Heap, stack, and allocators](week2/day13.md)
- [Day 14 — Week 2 review and interview drill](week2/day14.md)

### Week 3 — Concurrency
- [Day 15 — The concurrency problem](week3/day15.md)
- [Day 16 — Locks: mutex and spinlock](week3/day16.md)
- [Day 17 — Condition variables and semaphores](week3/day17.md)
- [Day 18 — Deadlock and lock ordering](week3/day18.md)
- [Day 19 — Memory models and atomics](week3/day19.md)
- [Day 20 — Signals and IPC](week3/day20.md)
- [Day 21 — Week 3 review and interview drill](week3/day21.md)

### Week 4 — I/O, filesystems, networking, synthesis
- [Day 22 — Files, inodes, directories](week4/day22.md)
- [Day 23 — The page cache and fsync](week4/day23.md)
- [Day 24 — Blocking, non-blocking, epoll](week4/day24.md)
- [Day 25 — Sockets and TCP](week4/day25.md)
- [Day 26 — How a packet flows](week4/day26.md)
- [Day 27 — Containers: namespaces and cgroups](week4/day27.md)
- [Day 28 — Observability: strace, perf, eBPF](week4/day28.md)
- [Day 29 — Full-stack interview drill](week4/day29.md)
- [Day 30 — Synthesis and what's next](week4/day30.md)

## A note on diagrams

Diagrams in these files use [Mermaid](https://mermaid.js.org/) syntax. They render automatically on GitHub, GitLab, Obsidian, VS Code with the Mermaid extension, and many other markdown viewers. If your viewer doesn't render Mermaid, the source is still readable as a text description.

Good luck. Patience and consistent daily effort beat heroic weekend marathons.
