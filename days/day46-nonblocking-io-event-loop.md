# Day 46: Non-blocking I/O & Event Loop

[← Back to Study Plan](../lld-study-plan.md) | [← Day 45](day45-sockets-tcp.md)

> **Time**: ~1.5-2 hours
> **Goal**: One thread per connection (Day 45) collapses at thousands of clients. The scalable answer is **non-blocking I/O** driven by an **event loop** — a single thread that watches many sockets and reacts only to the ones that are ready. Learn `O_NONBLOCK` and the `EAGAIN`/`EWOULDBLOCK` contract, the **reactor pattern**, fd multiplexing, the platform readiness APIs — `epoll` on Linux vs `kqueue` on macOS/BSD (and the portable-but-slow `poll`) — the crucial difference between **level-triggered and edge-triggered** notification, and why edge-triggered forces you to drain sockets until `EAGAIN`. Build a non-blocking echo server that serves many clients concurrently in one thread, with the platform multiplexer abstracted behind a thin interface.

---
---

# PART 1: WHY BLOCKING DOESN'T SCALE

---
---

<br>

<h2 style="color: #2980B9;">📘 46.1 The C10k Problem</h2>

The Day 45 server blocks in `recv` per client, so each concurrent connection needs its own thread:

```
  thread/conn model (10,000 clients):
  ┌────────┐┌────────┐┌────────┐         ┌────────┐
  │thread 1││thread 2││thread 3│  ...    │thread 10000│
  │recv(c1)││recv(c2)││recv(c3)│         │recv(c10000)│
  └────────┘└────────┘└────────┘         └────────┘
   each: ~1-8 MB stack + scheduler entry + context-switch cost
   → tens of GB of stacks, kernel drowning in context switches
```

Most of those threads are *asleep* in `recv` doing nothing — connections are mostly idle, waiting. We're paying for 10,000 threads to express "wake me when *any* of these sockets has data." That's exactly what a multiplexing syscall does in **one** thread:

```
  event-loop model (10,000 clients, 1 thread):
  ┌───────────────────────────────────────────────────┐
  │  while(true){ ready = wait_for_readiness(all_fds);  │
  │               for fd in ready: handle(fd); }        │
  └───────────────────────────────────────────────────┘
   one stack, no per-conn thread, scales to 10k+ on commodity hardware
```

This is how nginx, Redis, Node.js, and Envoy each handle tens of thousands of connections per thread.

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 46.2 O_NONBLOCK and the EAGAIN Contract</h2>

The enabler is putting sockets in **non-blocking mode**. A non-blocking `recv`/`send`/`accept` **never sleeps**: if it can't make progress immediately, it returns `-1` with `errno == EAGAIN` (== `EWOULDBLOCK` on Linux/macOS) instead of blocking the thread.

```cpp
int flags = ::fcntl(fd, F_GETFL, 0);
::fcntl(fd, F_SETFL, flags | O_NONBLOCK);   // make fd non-blocking
```

```
   Blocking recv on empty buffer:   thread SLEEPS until data
   Non-blocking recv on empty buf:  returns -1, errno = EAGAIN  ← "nothing now, ask later"
```

The new contract: `EAGAIN` is **not an error** — it means "no data/space right now; the event loop will tell you when to retry." Every read/write/accept loop must distinguish it:

```cpp
ssize_t n = ::recv(fd, buf, len, 0);
if (n > 0)                              { /* got data */ }
else if (n == 0)                        { /* EOF: peer closed */ }
else if (errno == EAGAIN)               { /* drained for now — return to loop */ }
else if (errno == EINTR)                { /* retry */ }
else                                    { /* real error: close fd */ }
```

So the only difference from Day 45's recv handling is the extra `EAGAIN` case — which means "stop reading and go back to waiting on the loop."

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 46.3 The Reactor Pattern</h2>

The event-loop architecture has a name: the **Reactor pattern**. A *demultiplexer* (the kernel's `epoll`/`kqueue`) waits on many fds; when some become ready, it dispatches to per-fd *event handlers*.

```
        ┌──────────────────── Reactor (one thread) ──────────────────────┐
        │                                                                  │
        │   register(fd, READ)                                             │
        │        │                                                         │
        │        ▼                                                         │
        │   ┌──────────────────┐   ready fds   ┌────────────────────────┐ │
        │   │  demultiplexer    │ ────────────► │ dispatch to handlers    │ │
        │   │ (epoll_wait /     │               │  listener fd → accept() │ │
        │   │  kevent)          │ ◄──────────── │  client fd  → recv/send │ │
        │   └──────────────────┘   re-arm       └────────────────────────┘ │
        └──────────────────────────────────────────────────────────────────┘
```

The handler does a small amount of non-blocking work and returns immediately to the loop — **it must never block.** One slow blocking call (a synchronous DB query, a blocking `recv`) freezes *every* connection, because there's only one thread. This "never block the loop" discipline is the core constraint of reactor-style servers.

The flow per iteration:
1. `wait` for readiness on all registered fds (the one place we're allowed to block).
2. For each ready fd: if it's the listener, `accept` (and register the new client); otherwise `recv`/`send` non-blocking until `EAGAIN`.
3. Repeat.

<br><br>

---
---

# PART 2: THE MULTIPLEXING APIS

---
---

<br>

<h2 style="color: #2980B9;">📘 46.4 poll vs epoll vs kqueue</h2>

Three generations of "tell me which fds are ready":

| API | Platform | Scaling | Notes |
|-----|----------|---------|-------|
| `select` | everywhere | O(n), fd ≤ 1024 | ancient, fd-set size limit, avoid |
| `poll` | POSIX (Linux + macOS) | O(n) per call | portable; you pass the *whole* fd array every call |
| `epoll` | **Linux only** | O(1) amortized | kernel keeps the set; you only get ready ones back |
| `kqueue` | **macOS / *BSD only** | O(1) amortized | same idea, different API; also handles timers, signals, file events |

`poll` is O(n): each call hands the kernel the entire list of fds and gets back the whole list with readiness flags — so cost grows with total connections even if only one is active. `epoll` and `kqueue` fix this: you register interest *once*, and each `wait` returns *only the fds that became ready* — O(number-of-ready), independent of total connections. That O(1)-per-event behavior is what makes 10k+ connections cheap.

Since the study environment is macOS but production is often Linux, we abstract both behind a tiny `Poller` interface and `#ifdef` the implementation. Same concept, two syscalls.

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 46.5 epoll (Linux) at a Glance</h2>

```cpp
int ep = ::epoll_create1(0);                       // create the epoll instance

epoll_event ev{};
ev.events  = EPOLLIN;                              // interested in "readable"
ev.data.fd = fd;
::epoll_ctl(ep, EPOLL_CTL_ADD, fd, &ev);           // register fd (once)

epoll_event events[64];
int n = ::epoll_wait(ep, events, 64, timeout_ms);  // block until some are ready
for (int i = 0; i < n; ++i) {
    int rfd = events[i].data.fd;                   // ONLY ready fds are returned
    // handle rfd ...
}
```

- Register/modify/remove with `epoll_ctl` + `EPOLL_CTL_ADD/MOD/DEL`.
- `EPOLLIN` (readable), `EPOLLOUT` (writable), `EPOLLET` (edge-triggered), `EPOLLRDHUP` (peer half-closed).
- `epoll_wait` returns only the fds that are ready — no scanning the whole set.

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 46.6 kqueue (macOS/BSD) at a Glance</h2>

```cpp
int kq = ::kqueue();                               // create the kqueue

struct kevent change;
EV_SET(&change, fd, EVFILT_READ, EV_ADD, 0, 0, nullptr);  // interested in readable
::kevent(kq, &change, 1, nullptr, 0, nullptr);     // register (changelist)

struct kevent events[64];
int n = ::kevent(kq, nullptr, 0, events, 64, /*timeout*/nullptr);  // block for ready
for (int i = 0; i < n; ++i) {
    int rfd = static_cast<int>(events[i].ident);   // ONLY ready fds
    if (events[i].flags & EV_EOF) { /* peer closed */ }
    // handle rfd ...
}
```

- `EV_SET` fills a `kevent`; `kevent()` is *both* the register call (changelist in) and the wait call (eventlist out).
- Filters: `EVFILT_READ`, `EVFILT_WRITE`, plus `EVFILT_TIMER`, `EVFILT_SIGNAL`, `EVFILT_VNODE` (file changes) — more general than epoll.
- `EV_ADD`/`EV_DELETE`/`EV_ENABLE`/`EV_DISABLE`; `EV_CLEAR` selects edge-triggered.

The mapping is direct: `epoll_create1` ↔ `kqueue`, `epoll_ctl(ADD)` ↔ `EV_SET(EV_ADD)`, `epoll_wait` ↔ `kevent`, `EPOLLET` ↔ `EV_CLEAR`.

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 46.7 Level-Triggered vs Edge-Triggered</h2>

The most error-prone concept in event loops. It governs *when* the multiplexer reports an fd as ready.

| | **Level-Triggered (LT)** | **Edge-Triggered (ET)** |
|---|---|---|
| Reports ready when | data is *present* (every wait, while readable) | data *arrives* (only on the transition) |
| If you read partially | reported again next `wait` | **not** reported again until *more* arrives |
| Programming model | read once per event, fine | **must drain until `EAGAIN`** every event |
| Risk | none re: missed data (just an extra wakeup) | miss data forever if you stop reading early |
| Default | `epoll` (LT), `kqueue` (LT) | opt-in: `EPOLLET` / `EV_CLEAR` |

```
   100 bytes arrive, you read 40:

   Level-triggered:                Edge-triggered:
     wait → ready (100 buffered)     wait → ready (edge: data arrived)
     read 40 (60 left)               read 40 (60 left)
     wait → READY AGAIN (60 left)    wait → NOT reported (no new arrival!)
     read 60 → drained               → 60 bytes stuck until MORE data arrives
                                        ⇒ you must loop read until EAGAIN
```

**Rule**: with edge-triggered, on every readiness event you *must* loop `recv` until it returns `EAGAIN`, and loop `send` until `EAGAIN` — otherwise buffered data/space goes unnoticed and the connection hangs. Level-triggered is more forgiving (read what you want; you'll be told again) but wakes you more often. ET reduces wakeups (good for high throughput) at the cost of the drain-completely discipline.

We'll build **edge-triggered** because it's the scalable default and forces the correct draining habit — and the drain loop also handles the byte-stream partial reads from Day 45 naturally.

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 46.8 Writability and the Output Buffer</h2>

`send` on a non-blocking socket can return `EAGAIN` when the kernel send buffer is full (a slow client). You can't block waiting for space — so the reactor pattern handles this with a **per-connection output buffer**:

```
  want to send 10 KB, kernel accepts 3 KB (then EAGAIN):
    1. send what you can (3 KB)
    2. stash the remaining 7 KB in conn.outBuf
    3. register interest in EPOLLOUT / EVFILT_WRITE for this fd
    4. when the loop reports the fd WRITABLE, flush more from outBuf
    5. when outBuf empties, stop watching for writable (avoid busy-wakeups)
```

This "watch for writable only while you have pending output" is standard reactor bookkeeping. For the echo server we keep it minimal (echo is small), but the pattern is essential for any real protocol with large or slow responses.

<br><br>

---
---

# PART 3: BUILD EXERCISE

---
---

<br>

<h2 style="color: #2980B9;">📘 46.9 Exercise: Single-Threaded Non-blocking Echo Server</h2>

Convert the Day 45 echo server to non-blocking + event loop, serving many clients in one thread. Abstract the multiplexer behind a `Poller` that compiles to `epoll` on Linux and `kqueue` on macOS. Use edge-triggered mode and drain each socket until `EAGAIN`.

> **Portability note**: The `Poller` class `#ifdef`s `__linux__` → epoll vs `__APPLE__`/BSD → kqueue. Both expose the same `add`/`del`/`wait` interface returning ready fds. Builds and runs on Linux and macOS.

<br>

#### Skeleton — `poller.h` (epoll/kqueue abstraction)

```cpp
// poller.h
#pragma once
#include <fcntl.h>
#include <unistd.h>
#include <vector>

inline void setNonBlocking(int fd) {
    int f = ::fcntl(fd, F_GETFL, 0);
    ::fcntl(fd, F_SETFL, f | O_NONBLOCK);
}

#if defined(__linux__)
#include <sys/epoll.h>

class Poller {
    int ep_;
public:
    Poller()  { ep_ = ::epoll_create1(0); }
    ~Poller() { ::close(ep_); }

    void add(int fd) {                                  // edge-triggered read interest
        epoll_event ev{};
        ev.events  = EPOLLIN | EPOLLET;
        ev.data.fd = fd;
        ::epoll_ctl(ep_, EPOLL_CTL_ADD, fd, &ev);
    }
    void del(int fd) { ::epoll_ctl(ep_, EPOLL_CTL_DEL, fd, nullptr); }

    // Fill `ready` with fds that became readable; returns count.
    int wait(std::vector<int>& ready) {
        epoll_event evs[64];
        int n = ::epoll_wait(ep_, evs, 64, -1);
        ready.clear();
        for (int i = 0; i < n; ++i) ready.push_back(evs[i].data.fd);
        return n;
    }
};

#elif defined(__APPLE__) || defined(__FreeBSD__)
#include <sys/event.h>

class Poller {
    int kq_;
public:
    Poller()  { kq_ = ::kqueue(); }
    ~Poller() { ::close(kq_); }

    void add(int fd) {                                  // edge-triggered (EV_CLEAR) read
        struct kevent ch;
        EV_SET(&ch, fd, EVFILT_READ, EV_ADD | EV_CLEAR, 0, 0, nullptr);
        ::kevent(kq_, &ch, 1, nullptr, 0, nullptr);
    }
    void del(int fd) {
        struct kevent ch;
        EV_SET(&ch, fd, EVFILT_READ, EV_DELETE, 0, 0, nullptr);
        ::kevent(kq_, &ch, 1, nullptr, 0, nullptr);
    }

    int wait(std::vector<int>& ready) {
        struct kevent evs[64];
        int n = ::kevent(kq_, nullptr, 0, evs, 64, nullptr);
        ready.clear();
        for (int i = 0; i < n; ++i) ready.push_back(static_cast<int>(evs[i].ident));
        return n;
    }
};
#else
#error "Unsupported platform: need epoll (Linux) or kqueue (macOS/BSD)"
#endif
```

<br>

#### Skeleton — `nb_echo_server.cpp`

```cpp
// nb_echo_server.cpp
#include "poller.h"
#include <arpa/inet.h>
#include <cerrno>
#include <csignal>
#include <cstring>
#include <iostream>
#include <netinet/in.h>
#include <set>
#include <sys/socket.h>

static int makeListener(uint16_t port) {
    int lfd = ::socket(AF_INET, SOCK_STREAM, 0);
    int yes = 1;
    ::setsockopt(lfd, SOL_SOCKET, SO_REUSEADDR, &yes, sizeof yes);
    sockaddr_in a{};
    a.sin_family = AF_INET; a.sin_addr.s_addr = htonl(INADDR_ANY); a.sin_port = htons(port);
    if (::bind(lfd, reinterpret_cast<sockaddr*>(&a), sizeof a) < 0) { perror("bind"); _exit(1); }
    if (::listen(lfd, 128) < 0) { perror("listen"); _exit(1); }
    setNonBlocking(lfd);                       // listener is non-blocking too
    return lfd;
}

// Edge-triggered: accept ALL pending clients until EAGAIN.
static void acceptAll(int lfd, Poller& poller, std::set<int>& clients) {
    for (;;) {
        int cfd = ::accept(lfd, nullptr, nullptr);
        if (cfd >= 0) {
            setNonBlocking(cfd);
            poller.add(cfd);
            clients.insert(cfd);
            std::cout << "accepted fd " << cfd << " (" << clients.size() << " clients)\n";
        } else if (errno == EAGAIN || errno == EWOULDBLOCK) {
            break;                             // drained the accept queue
        } else if (errno == EINTR) {
            continue;
        } else { perror("accept"); break; }
    }
}

// Edge-triggered: drain reads until EAGAIN; echo each chunk straight back.
// Returns false if the client should be closed (EOF or error).
static bool serviceClient(int cfd) {
    char buf[4096];
    for (;;) {
        ssize_t n = ::recv(cfd, buf, sizeof buf, 0);
        if (n > 0) {
            // Echo. (Simplified: assume the small echo fits the send buffer.
            //  Real code would buffer leftovers + watch for writable — see §46.8.)
            size_t off = 0;
            while (off < static_cast<size_t>(n)) {
                ssize_t w = ::send(cfd, buf + off, n - off, 0);
                if (w > 0) { off += static_cast<size_t>(w); continue; }
                if (w < 0 && (errno == EAGAIN || errno == EWOULDBLOCK)) {
                    // send buffer full; for the exercise we just spin briefly.
                    // (Proper fix: stash leftover, register EPOLLOUT/EVFILT_WRITE.)
                    continue;
                }
                if (w < 0 && errno == EINTR) continue;
                return false;                  // write error
            }
        } else if (n == 0) {
            return false;                      // EOF: peer closed
        } else if (errno == EAGAIN || errno == EWOULDBLOCK) {
            return true;                       // drained for now — keep the client
        } else if (errno == EINTR) {
            continue;
        } else {
            return false;                      // real recv error
        }
    }
}

int main(int argc, char** argv) {
    std::signal(SIGPIPE, SIG_IGN);
    uint16_t port = (argc > 1) ? static_cast<uint16_t>(std::stoi(argv[1])) : 8080;

    int lfd = makeListener(port);
    Poller poller;
    poller.add(lfd);
    std::set<int> clients;
    std::cout << "non-blocking echo server on port " << port << "\n";

    std::vector<int> ready;
    for (;;) {
        poller.wait(ready);                    // the ONLY blocking call
        for (int fd : ready) {
            if (fd == lfd) {
                acceptAll(lfd, poller, clients);
            } else if (!serviceClient(fd)) {
                poller.del(fd);
                ::close(fd);
                clients.erase(fd);
                std::cout << "closed fd " << fd << " (" << clients.size() << " clients)\n";
            }
        }
    }
}
```

<br>

#### Compile and run

```bash
# Linux (epoll) or macOS (kqueue) — same command, Poller picks the impl:
g++ -std=c++17 -Wall -Wextra -Wpedantic -o nb_echo_server nb_echo_server.cpp

./nb_echo_server 8080
```

<br>

#### Drive it with multiple clients

```bash
# Reuse the Day 45 client, or fire several netcat sessions at once:
( printf 'one\n';   sleep 2 ) | nc 127.0.0.1 8080 &
( printf 'two\n';   sleep 2 ) | nc 127.0.0.1 8080 &
( printf 'three\n'; sleep 2 ) | nc 127.0.0.1 8080 &
wait
```

<br>

#### Expected output pattern

```
non-blocking echo server on port 8080
accepted fd 5 (1 clients)
accepted fd 6 (2 clients)
accepted fd 7 (3 clients)        ← all three served concurrently by ONE thread
closed fd 5 (2 clients)
closed fd 6 (1 clients)
closed fd 7 (0 clients)
```

All three connections are handled simultaneously by a single thread — no per-client thread, no blocking. Each `nc` receives its line echoed back.

<br>

#### Bonus Challenges

1. **Prove edge-triggered draining is required**: in `serviceClient`, read only *once* per event (remove the `for(;;)` drain). Send a large burst and watch data get stuck (the connection stalls) — then restore the loop and confirm it works. This is the canonical ET bug.
2. **Output buffering done right**: implement §46.8 — give each connection a `std::string outBuf`; on `EAGAIN` from `send`, stash the remainder, register write interest (`EPOLLOUT`/`EVFILT_WRITE`), flush on the writable event, and deregister write interest when drained. Test with a deliberately slow-reading client.
3. **Level-triggered variant**: switch to LT (drop `EPOLLET`/`EV_CLEAR`), read once per event, and observe that it still works but the loop wakes more often. Compare wakeup counts.
4. **Add a timer**: integrate a per-connection idle timeout. On Linux use `timerfd` registered in epoll; on macOS use `EVFILT_TIMER` in the same kqueue. Close connections idle > 30 s.
5. **Self-pipe / eventfd wakeup**: add a way for another thread to wake the loop (to inject work or signal shutdown) — the "self-pipe trick" (portable) or Linux `eventfd`.
6. **Benchmark**: compare this single-threaded server vs the Day 45 thread-per-connection server under 5,000 concurrent idle connections. Measure memory (RSS) and CPU.

<br><br>

---
---

# PART 4: DEEP DIVE — COMMON QUESTIONS

---
---

<br>

<h2 style="color: #2980B9;">📘 46.10 Q&A</h2>

<br>

#### Q1: "What does `EAGAIN`/`EWOULDBLOCK` mean and why isn't it an error?"

On a non-blocking socket it means "this operation can't make progress *right now* without blocking, so I returned instead of sleeping." For `recv` it means the receive buffer is empty; for `send`, the send buffer is full; for `accept`, no pending connections. It's the normal signal that you've drained the socket — you stop and return to the event loop, which will tell you when the fd is ready again. Treating it as an error (closing the connection) is a common bug.

<br>

#### Q2: "epoll vs kqueue vs poll — which and why?"

`poll` is portable but O(n): every call re-scans all fds, so cost grows with total connections. `epoll` (Linux) and `kqueue` (macOS/BSD) register interest once and return only the fds that became ready — O(number-of-events), independent of total connections — which is what scales to 10k+. They're the same idea with different syscalls, so abstract them behind one interface and `#ifdef` the implementation. Use `poll` only for small fd counts or maximum portability.

<br>

#### Q3: "Explain level-triggered vs edge-triggered."

Level-triggered reports an fd ready *whenever* it's readable — so if you read part of the buffered data, the next `wait` reports it ready again. Edge-triggered reports only on the *transition* (when new data arrives), so it won't re-notify just because data is still buffered. Therefore with edge-triggered you **must drain the socket fully (loop until `EAGAIN`)** on every event, or buffered bytes sit unnoticed until more arrive and the connection appears to hang. ET means fewer wakeups but stricter rules; LT is more forgiving but wakes more often.

<br>

#### Q4: "What's the one rule I must never break in an event loop?"

Never block the loop. There's a single thread, so any blocking call — a synchronous DB query, a blocking `recv`, a long compute, a mutex held by a slow path — freezes *every* connection until it returns. All I/O must be non-blocking, all handlers must do small bounded work and return quickly, and anything genuinely slow must be offloaded to a worker thread/pool with the result delivered back via a wakeup (eventfd/self-pipe).

<br>

#### Q5: "How do I handle a `send` that returns `EAGAIN`?"

The kernel send buffer is full (slow/backed-up client). You can't block, so: send what was accepted, stash the unsent remainder in a per-connection output buffer, and register write interest (`EPOLLOUT`/`EVFILT_WRITE`) for that fd. When the loop reports the fd writable, flush more from the buffer; once it's empty, deregister write interest (otherwise you get a storm of writable events). This back-pressure handling is mandatory for any protocol with responses larger than the send buffer.

<br>

#### Q6: "Why must the listening socket also be non-blocking, and why accept in a loop?"

With edge-triggered notification, the listener is reported ready *once* when connections arrive — even if several arrived together. If you `accept` only one per event, the rest sit in the accept queue unnoticed until the next arrival. So you loop `accept` until it returns `EAGAIN`, draining the whole accept queue. The listener must be non-blocking for that loop to terminate instead of blocking on the final (empty) `accept`.

<br>

#### Q7: "Is the reactor pattern the same as `async`/`await` or coroutines?"

They're layers. The reactor (epoll/kqueue loop) is the low-level engine. `async`/`await`, coroutines (C++20), and futures are *programming models* built on top: the runtime registers fds with a reactor and resumes your coroutine when the fd is ready, so your code reads sequentially while the machinery underneath is the same non-blocking event loop. Node's libuv, Rust's tokio, and Boost.Asio all wrap epoll/kqueue this way.

<br>

#### Q8: "How do I scale a single-threaded reactor across CPU cores?"

A single reactor uses one core. To use N cores, run N event loops (one per thread), each with its own epoll/kqueue. Distribute connections across them either by having each thread `accept` from the same listener (with `SO_REUSEPORT` so the kernel load-balances accepts) or by a single acceptor thread that hands new fds to worker loops round-robin. This "multiple reactors" model is what nginx (worker processes) and Netty (event-loop group) use.

<br><br>

---

## Reflection Questions

1. Why does one-thread-per-connection collapse at 10k clients, and how does a single-threaded event loop express the same intent more cheaply?
2. What is the `EAGAIN` contract on a non-blocking socket, and how does it change your recv/send/accept loops vs Day 45?
3. Draw the reactor pattern. What is the one rule a handler must never violate, and why?
4. Why is `epoll`/`kqueue` O(1) per event while `poll` is O(n)? What changes between them and `select`?
5. Explain level- vs edge-triggered. Why does edge-triggered force you to drain until `EAGAIN`?
6. When `send` returns `EAGAIN`, what bookkeeping does the reactor do to deliver the rest later?

---

## Interview Questions

1. "Why doesn't thread-per-connection scale, and what's the alternative?"
2. "What is `O_NONBLOCK`, and what does `EAGAIN` tell you?"
3. "Explain the reactor pattern. What must an event handler never do?"
4. "Compare `select`, `poll`, `epoll`, and `kqueue`. Why is epoll/kqueue faster at scale?"
5. "Level-triggered vs edge-triggered — explain the difference and the bug edge-triggered invites."
6. "With edge-triggered epoll, why must you loop `recv`/`accept` until `EAGAIN`?"
7. "A `send` returns `EAGAIN` because the client is slow. How does your server handle the unsent data?"
8. "How would you make a single-threaded event-loop server use all CPU cores?"
9. "How do `async`/`await` and coroutines relate to an epoll/kqueue event loop?"
10. "Sketch a portable `Poller` abstraction that uses epoll on Linux and kqueue on macOS."

---

**Next**: Day 47 — Serialization →
