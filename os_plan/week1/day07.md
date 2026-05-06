# Day 7 — Week 1 review and interview drill

> **Week 1 · Foundations**
> No new reading. Today is consolidation.

## Why this matters

Reviewing matters more than learning new things. Spaced repetition and explicit articulation — out loud — are how knowledge moves from short-term to long-term memory. Today we cement Week 1 by stress-testing your ability to talk through processes, threads, and scheduling fluently.

## 7.1 Mental model check

Before reading further, try writing out (on paper, or out loud) answers to these. Don't peek.

1. What's a process? What's a thread? What does a `task_struct` contain?
2. What does `fork()` do, step by step? What does `exec()` do?
3. What's a context switch and what does it cost?
4. What's the difference between user and kernel mode?
5. How does CFS work, in two minutes?

Now read on and compare.

## 7.2 The unifying picture

Everything we covered Week 1 is connected:

```mermaid
graph TB
    Hardware["Hardware: CPUs, memory, devices"]
    Kernel["Kernel<br/>(ring 0, privileged)"]

    Hardware --> Kernel

    Kernel --> Tasks["task_struct (PID 0..N)"]
    Tasks --> P1["Process<br/>own mm, own files"]
    Tasks --> T1["Thread<br/>shared mm, shared files"]

    Kernel --> Sched["Scheduler<br/>CFS / RT / deadline"]
    Sched --> CPU["Per-CPU run queue<br/>rb-tree by vruntime"]

    P1 -.->|fork()| P2[New process]
    P1 -.->|pthread_create / clone| T2[New thread]
    P1 -.->|execve()| P1prime[Same PID, new program]

    style Kernel fill:#1F3864,color:#fff
    style Sched fill:#2E75B6,color:#fff
```

The kernel sits between hardware and processes. Each process is a `task_struct`. The scheduler picks which task runs on each CPU. Process creation goes through `clone()` (or its `fork()` variant). Program replacement goes through `execve()`. All sharing patterns (process vs. thread) are just `clone()` flag combinations.

If you can draw and explain this diagram from memory, your foundations are solid.

## 7.3 Common interview traps and clean answers

**Trap 1**: "Is `fork` expensive?" The naive answer "yes, it copies everything" is wrong post-1990. Correct answer: page tables are duplicated but pages are COW-shared, so it's cheap (microseconds for a multi-GB process), and most fork+exec patterns never copy anything. The expensive part on huge processes is the page table duplication itself.

**Trap 2**: "Threads share the stack." Wrong — they share the *address space*, but each thread has its own stack region within it. Threads share the heap, globals, code, mmap'd memory. Each has its own stack, registers, errno, signal mask.

**Trap 3**: "Higher priority means runs first." Only for real-time scheduling. In CFS (SCHED_OTHER), priority (via nice) means more CPU share over time, not strict precedence. Both a nice-0 and a nice+10 task will run; the nice-0 just runs more often.

**Trap 4**: "If a process exits, its children die." Wrong. Children get reparented to PID 1 (init/systemd), which reaps them. To kill children when parent exits, you have to be explicit (process groups + `kill(-pgid, SIGTERM)`, or `prctl(PR_SET_PDEATHSIG, SIGTERM)` in each child).

**Trap 5**: "epoll uses threads." No — `epoll` lets a single thread monitor many fds. We'll cover this Day 24, but it's a common confusion to flag now.

## 7.4 A long-form interview answer to rehearse

**Question**: "Walk me through what happens when a user types `ls | wc -l` in a shell."

**Model answer** (about 3 minutes spoken):

> The shell parses the command line into a pipeline of two commands: `ls` and `wc -l`. It needs to set up a pipe between them and run both processes simultaneously.
>
> First, the shell calls `pipe()` — a syscall that creates a unidirectional in-kernel buffer with two file descriptors: one for reading, one for writing. Say the kernel returns fds 3 and 4, with 3 being read and 4 being write.
>
> Next, the shell forks twice. The first child will become `ls`. Inside that child, the shell sets up file descriptors: it calls `dup2(4, 1)` to make fd 4 (the pipe write end) become stdout. Then it closes the original fd 4 and the read end fd 3 — they're not needed in `ls`. Now it calls `execve("/bin/ls", ...)` and `ls` begins running with stdout connected to the pipe.
>
> The second child becomes `wc`. It does `dup2(3, 0)` to make the pipe read end become stdin, closes the originals, and execs `/usr/bin/wc -l`.
>
> Back in the shell, it closes both pipe fds (it doesn't need them) and calls `wait` to reap both children. As `ls` writes its output, the kernel buffers it in the pipe; `wc` reads from the buffer; if the buffer fills, `ls` is blocked until `wc` reads more. When `ls` finishes, it closes the pipe write end; `wc` reads EOF and finishes too. The shell collects the exit status of both, and prints the prompt again.
>
> Underneath, the kernel was busy: it was scheduling the three processes (shell, `ls`, `wc`) on whatever CPUs were available, copying-on-write each forked process's address space, transitioning between user and kernel mode for each `pipe`, `dup2`, `read`, and `write` syscall, and managing the pipe's circular buffer.

If you can give that answer fluently, you're solidly into Week 2 territory.

## 7.5 Mock interview drill (60 minutes)

Set a timer. Answer each out loud, ideally to someone else but recording yourself works. After each, score honestly: confident / hesitant / wrong.

### Section A: definitions and concepts (target: 1 min each)

1. What's the difference between a process and a thread on Linux?
2. What's a system call? Why is it a privileged operation?
3. What's a context switch? When does one happen?
4. What's a zombie process?
5. What does the `nice` value do?

### Section B: walkthroughs (target: 3 min each)

1. Walk me through `fork()` — what does the kernel actually do?
2. Walk me through `execve()` — what survives, what doesn't?
3. Walk me through how CFS picks the next task to run.
4. Walk me through `pipe()` — how does data flow from writer to reader?
5. Walk me through what happens when the timer interrupt fires while a user process is running.

### Section C: design and reasoning (target: 5 min each)

1. You have a server that needs to handle 10,000 concurrent connections. What's wrong with one thread per connection? What are the alternatives? (We'll cover details Day 24.)
2. A process has been in `D` state for 5 minutes. How would you investigate?
3. You see `cs` in `vmstat` is 200,000/sec on a 4-CPU server. Is that bad? How would you find the cause?
4. A multi-threaded program calls `fork()`. What can go wrong? How do you mitigate?
5. Design a small program that demonstrates priority inversion (no need to write code; just describe the structure).

## 7.6 Where you should be by end of Week 1

- You can explain process vs. thread to a non-expert without confusing yourself.
- You can describe COW fork without saying anything that's outright wrong.
- You can walk through a shell pipeline end to end.
- You know the difference between SCHED_OTHER, SCHED_FIFO, SCHED_RR, SCHED_DEADLINE.
- You can explain CFS at a high level (vruntime, rb-tree, fair sharing).
- You're comfortable inspecting tasks via `/proc`, `ps -eLf`, `top -H`.

## 7.7 Sample model answers to Section A

**Q1. Process vs. thread**: Both are tasks (`task_struct`s) in Linux. A process has its own address space; threads share their parent's address space, file descriptors, and signal handlers. Each thread has its own stack, registers, and TID. Process creation duplicates more state (cheap due to COW); thread creation is faster. Communication between threads is just memory access; between processes requires explicit IPC (pipes, shared memory, etc.).

**Q2. System call**: A controlled transition from user mode to kernel mode. The user process places arguments in registers and executes the `syscall` instruction (x86-64). The CPU switches to ring 0, jumps to the kernel's syscall entry, the kernel validates the request, performs the privileged operation (file I/O, network, memory mapping), and returns. Privileged because the kernel can do things a user process can't — talk to hardware, change page tables, send arbitrary signals. Without this controlled gate, security and isolation fall apart.

**Q3. Context switch**: The kernel saves one task's CPU state to its `task_struct` and restores another task's state, then resumes execution at the new task's instruction pointer. Triggered by: timer interrupt (preemption), voluntary block (waiting for I/O), syscall return that the scheduler decides to redirect, or task exit. Cost is 1–10 µs direct, plus indirect costs from cache and TLB invalidation when switching across address spaces.

**Q4. Zombie**: A process that has called `exit()` but whose parent hasn't called `wait()` yet to collect the exit status. The kernel keeps a minimal `task_struct` containing the exit code; the process is gone for all practical purposes. Once parent waits, the entry is freed. Zombies persist in `ps` as `Z` state.

**Q5. Nice**: An adjustment to scheduling weight in CFS. Range −20 to +19, default 0. Lower nice = more CPU share. Each step is roughly 1.25× weight, so nice 0 vs. nice 10 gives the nice-0 task ~9× more CPU when both are competing. Doesn't affect scheduling class — both are still SCHED_OTHER.

## 7.8 Where to focus revisions

If your scoring on the drill above showed weak spots:

- **fork/exec details fuzzy** → re-read Day 3, write a small fork/exec/wait program.
- **Threads vs. processes mixed up** → re-read Day 4, run `ps -eLf` and trace what's there.
- **Scheduling abstract** → re-read Days 5–6, look at `/proc/sched_debug` and your shell's `/proc/$$/sched`.
- **Can't describe a context switch** → trace one with `perf sched record` then `perf sched latency`.

Be honest. Better to spend an extra day on Week 1 than to march into Week 2 with shaky foundations.

## 7.9 What's next

Week 2 is memory. We'll cover:
- Virtual address spaces and the user-process layout
- Paging and the TLB (the most-asked memory topic in interviews)
- Page tables in detail
- Page faults — minor, major, COW
- mmap and shared memory
- The heap, the stack, and how `malloc` works

Memory is the second-most-asked-about topic in systems interviews after concurrency. Most candidates say "virtual memory translates virtual to physical." Week 2 will get you to "how, exactly, including TLB, page tables levels, faults, COW, demand paging, and how the heap actually grows." That's the level of depth that distinguishes senior candidates.

## 7.10 Reflection

Before moving on, take 10 minutes and write (briefly):

- One concept from Week 1 you understand much better than before.
- One concept that still feels wobbly.
- One thing you're curious to dig deeper into when time allows.

Self-knowledge about your own understanding is the senior-engineer skill that compounds.
