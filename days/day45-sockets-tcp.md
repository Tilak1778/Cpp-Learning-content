# Day 45: Sockets & TCP Basics

[← Back to Study Plan](../lld-study-plan.md) | [← Day 44](day44-kv-store.md)

> **Time**: ~1.5-2 hours
> **Goal**: The Berkeley sockets API is the foundation of all network programming on Unix — every HTTP server, database client, and message broker bottoms out in `socket`/`bind`/`listen`/`accept`/`connect`/`send`/`recv`. Learn the server and client call sequences, what each call actually does, **blocking I/O** semantics, why `SO_REUSEADDR` exists (the `TIME_WAIT` "address already in use" trap), the all-important fact that TCP is a **byte stream not a message stream** (so reads and writes are *partial* — you must loop), and the precise return-value semantics of `recv`/`send` (0 = EOF, -1 = error, EINTR, EPIPE). Build a blocking TCP echo server and a matching client, written portably for Linux and macOS.

---
---

# PART 1: WHAT A SOCKET IS

---
---

<br>

<h2 style="color: #2980B9;">📘 45.1 The Socket Abstraction</h2>

A socket is a **file descriptor** that represents one endpoint of a network connection. Because it's an fd, it plugs into the same `read`/`write`/`close`/`poll` machinery as files and pipes — "everything is a file" is what makes Unix networking composable.

```
   process A (server)                     process B (client)
   ┌──────────────────┐                   ┌──────────────────┐
   │  fd 4  ◄────────┐ │                   │ ┌──────► fd 3     │
   └──────────────────┘ │                  │ │                │
                        │                  │ │                │
              kernel TCP stack  ◄──────────┼─┘  kernel TCP stack
              (send/recv buffers)          │    (send/recv buffers)
                        └──────  TCP segments over IP  ──────┘
```

Each socket has a kernel-side **send buffer** and **receive buffer**. `send()` copies your bytes into the send buffer (the kernel transmits them asynchronously); `recv()` copies bytes out of the receive buffer. Your application never touches the wire directly — it talks to these buffers through the fd.

Key socket attributes:
- **Domain**: `AF_INET` (IPv4), `AF_INET6` (IPv6), `AF_UNIX` (local IPC).
- **Type**: `SOCK_STREAM` (TCP — reliable, ordered, connection-oriented) vs `SOCK_DGRAM` (UDP — best-effort datagrams). We use TCP.

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 45.2 The Server Call Sequence</h2>

A TCP server runs five syscalls in order. Each does one specific thing:

```
  socket()  ── create an endpoint fd (unbound, not yet a server)
     │
  setsockopt(SO_REUSEADDR)  ── allow immediate re-bind (see §45.5)
     │
  bind()    ── attach the fd to a local (IP, port), e.g. 0.0.0.0:8080
     │
  listen()  ── mark it passive: kernel starts accepting connections,
     │          queuing completed handshakes (backlog)
     │
  accept()  ── BLOCK until a client connects; return a NEW fd for THAT client
     │          (the listening fd stays open to accept more)
     ▼
  recv()/send() on the per-client fd ... then close() it.
```

The crucial distinction: `listen` fd is the "doorbell" (it never carries data), `accept` returns a fresh "conversation" fd per client. A server loops on `accept`, spawning work per connected socket.

```cpp
int lfd = ::socket(AF_INET, SOCK_STREAM, 0);
int yes = 1;
::setsockopt(lfd, SOL_SOCKET, SO_REUSEADDR, &yes, sizeof yes);

sockaddr_in addr{};
addr.sin_family = AF_INET;
addr.sin_addr.s_addr = htonl(INADDR_ANY);   // bind all interfaces (0.0.0.0)
addr.sin_port = htons(8080);                 // network byte order!

::bind(lfd, reinterpret_cast<sockaddr*>(&addr), sizeof addr);
::listen(lfd, /*backlog=*/128);

for (;;) {
    int cfd = ::accept(lfd, nullptr, nullptr);   // blocks until a client arrives
    handle(cfd);                                  // talk to this client
    ::close(cfd);
}
```

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 45.3 The Client Call Sequence</h2>

A client is simpler — no `bind`/`listen`/`accept`, just `socket` then `connect`:

```
  socket()  ── create an endpoint fd
     │
  connect() ── BLOCK while the kernel performs the 3-way handshake
     │          to the server's (IP, port)
     ▼
  send()/recv() ... close()
```

```cpp
int fd = ::socket(AF_INET, SOCK_STREAM, 0);
sockaddr_in srv{};
srv.sin_family = AF_INET;
srv.sin_port   = htons(8080);
::inet_pton(AF_INET, "127.0.0.1", &srv.sin_addr);   // text IP → binary
::connect(fd, reinterpret_cast<sockaddr*>(&srv), sizeof srv);
// now send()/recv()
```

`connect()` triggers the SYN → SYN-ACK → ACK handshake and returns once the connection is ESTABLISHED (in blocking mode). Note `inet_pton` ("presentation to network") parses a dotted-quad string into the binary address; its inverse is `inet_ntop`.

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 45.4 The Handshake and the Two Queues</h2>

`listen(fd, backlog)` sets up the kernel's accept machinery. Behind the scenes there are **two queues**:

```
  client SYN ──►  ┌─────────────────┐  handshake completes  ┌──────────────────┐
                  │  SYN queue       │ ───────────────────►  │  accept queue     │
                  │ (half-open,      │  (SYN-ACK / ACK done) │ (ESTABLISHED,     │
                  │  awaiting ACK)   │                       │  ready for accept)│
                  └─────────────────┘                        └──────────────────┘
                                                                      │
                                                              accept() pops one
```

- **SYN queue**: connections mid-handshake (received SYN, sent SYN-ACK, awaiting the final ACK).
- **Accept queue**: fully-established connections waiting for the application to `accept()` them. `backlog` bounds this queue.

If the app is slow to `accept` and the accept queue fills, new connections are dropped/refused — the client sees a timeout or `ECONNREFUSED`. A too-small backlog under load causes mysterious connection failures; 128–1024 is typical for busy servers.

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 45.5 SO_REUSEADDR — the TIME_WAIT Trap</h2>

Restart a server you just killed and you'll often hit:

```
bind: Address already in use   (errno EADDRINUSE)
```

even though no process is using the port. The cause is TCP **`TIME_WAIT`**: when *your* server closes a connection, that socket lingers in `TIME_WAIT` for ~2×MSL (up to ~60s) to absorb any straggling packets from the old connection. While a socket associated with the port sits in `TIME_WAIT`, the kernel refuses to `bind` to it again.

`SO_REUSEADDR` tells the kernel "let me bind even though there are `TIME_WAIT` sockets on this port" — it's safe and effectively mandatory for any server you'll restart:

```cpp
int yes = 1;
::setsockopt(lfd, SOL_SOCKET, SO_REUSEADDR, &yes, sizeof yes);  // BEFORE bind()
```

Must be set *before* `bind()`. (`SO_REUSEPORT`, a different option, lets *multiple* sockets bind the same port for load-balanced accept across threads — useful later, but not the same thing.)

```
  Without SO_REUSEADDR:           With SO_REUSEADDR:
   old conn → TIME_WAIT (60s)      old conn → TIME_WAIT (60s)
   restart → bind() FAILS          restart → bind() SUCCEEDS immediately
```

<br><br>

---
---

# PART 2: THE BYTE-STREAM REALITY

---
---

<br>

<h2 style="color: #2980B9;">📘 45.6 TCP Is a Byte Stream, Not a Message Stream</h2>

This is the single most important and most-violated fact about TCP. **TCP does not preserve message boundaries.** If the client calls `send("HELLO")` then `send("WORLD")`, the server's `recv()` calls may return:

- `"HELLOWORLD"` (coalesced into one read), or
- `"HEL"`, then `"LOWORLD"` (split arbitrarily), or
- `"HELLO"`, then `"WORLD"` (matching the sends — but you can't rely on it), or
- any other split.

```
  app sends:   [ HELLO ][ WORLD ]     ← logical messages (in your head)
  on the wire: H E L L O W O R L D     ← TCP sees ONE undelimited byte stream
  app recvs:   [ HE ][ LLOWOR ][ LD ]  ← whatever the kernel buffers happened to hold
```

The kernel batches and splits based on buffer state, MTU, Nagle's algorithm, and timing — none of which your application controls. **A single `recv()` returns "some bytes that have arrived," not "the message the peer sent."**

The consequence: to send messages over TCP you need a **framing** protocol — length prefixes, delimiters, or fixed sizes — to recover boundaries the stream destroyed. That's the entire subject of Day 47. For today's echo server we sidestep it by just echoing whatever bytes arrive, in whatever chunks.

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 45.7 recv() Return Semantics — the Three Cases</h2>

Every `recv()` (and `read()`) on a socket returns one of three things. Mishandling these is the #1 source of socket bugs:

```cpp
ssize_t n = ::recv(fd, buf, len, 0);
if (n > 0) {
    // n bytes received (n MAY be < len — that's normal, not an error)
} else if (n == 0) {
    // ORDERLY SHUTDOWN: peer called close()/shutdown(). EOF. No more data ever.
    // Action: close(fd) and stop reading.
} else { // n < 0
    // ERROR. Inspect errno:
    //   EINTR        → interrupted by a signal; just retry the recv
    //   EAGAIN/EWOULDBLOCK → (non-blocking only) no data right now; try later
    //   ECONNRESET   → peer sent RST (crashed/aborted); treat as dead
    //   others       → real error; close and report
}
```

| Return | Meaning | What to do |
|--------|---------|------------|
| `> 0` | got *n* bytes (possibly fewer than asked) | process them; loop for more if needed |
| `== 0` | peer closed cleanly (FIN) — **EOF** | close your fd; connection over |
| `< 0` + `EINTR` | signal interrupted the call | retry |
| `< 0` + `EAGAIN` | (non-blocking) nothing available now | come back later (Day 46) |
| `< 0` + other | genuine error (`ECONNRESET`, ...) | close, log |

The classic bug: treating `recv` returning fewer bytes than requested as an error, or forgetting that `0` means EOF (treating it as "got zero bytes, keep looping" → spin forever).

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 45.8 send() Is Also Partial — Loop Until Done</h2>

Symmetrically, `send()` may accept **fewer bytes than you gave it** when the kernel send buffer is nearly full. It returns the count actually queued. You must loop:

```cpp
// Robust "send everything" helper.
bool sendAll(int fd, const char* data, size_t len) {
    size_t sent = 0;
    while (sent < len) {
        ssize_t n = ::send(fd, data + sent, len - sent, MSG_NOSIGNAL);
        if (n > 0) { sent += static_cast<size_t>(n); continue; }
        if (n < 0 && errno == EINTR) continue;     // retry
        return false;                               // EPIPE/ECONNRESET/etc.
    }
    return true;
}
```

Two gotchas:

1. **`MSG_NOSIGNAL`**: if you `send()` to a socket whose peer has closed, the kernel raises `SIGPIPE`, which by default **kills your process**. `MSG_NOSIGNAL` (Linux) suppresses it, returning `EPIPE` instead. macOS lacks `MSG_NOSIGNAL` on `send` but supports the `SO_NOSIGPIPE` socket option, or you can `signal(SIGPIPE, SIG_IGN)` globally — the portable choice.
2. **Short writes are normal**, not errors. The loop is mandatory for correctness under load; "I called send once with the whole buffer" works in toy programs and fails in production.

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 45.9 Blocking I/O and Its Limit</h2>

By default sockets are **blocking**: `accept`, `connect`, `recv`, and `send` suspend the calling thread until they can make progress.

```
  recv(fd) on empty buffer  →  thread SLEEPS until bytes arrive (or EOF/error)
  send(fd) on full buffer   →  thread SLEEPS until buffer drains
  accept(lfd) no clients    →  thread SLEEPS until a client connects
```

This is simple to reason about and perfect for the echo server below. Its limitation: **one blocked thread per connection.** A `recv` that blocks can't also watch other sockets. To serve many clients with blocking I/O you need one thread (or process) per connection — fine for dozens, expensive at thousands (thread stacks, context switches, the C10k problem). The escape is non-blocking I/O + an event loop, which is exactly Day 46.

<br><br>

---
---

# PART 3: BUILD EXERCISE

---
---

<br>

<h2 style="color: #2980B9;">📘 45.10 Exercise: Blocking TCP Echo Server + Client</h2>

Build a blocking echo server that accepts one client at a time, reads bytes, and writes them straight back (until the client closes), plus a client that connects, sends a few messages, and prints the echoes. This cements `socket`/`bind`/`listen`/`accept`/`connect`, the recv/send loops, partial-I/O handling, and `SO_REUSEADDR`.

> **Portability note**: Pure POSIX (`<sys/socket.h>`, `<netinet/in.h>`, `<arpa/inet.h>`, `<unistd.h>`). Builds and runs on Linux and macOS. We `signal(SIGPIPE, SIG_IGN)` for portable SIGPIPE handling instead of `MSG_NOSIGNAL` (which macOS lacks on `send`).

<br>

#### Skeleton — `net_util.h`

```cpp
// net_util.h
#pragma once
#include <arpa/inet.h>
#include <cerrno>
#include <cstring>
#include <netinet/in.h>
#include <stdexcept>
#include <string>
#include <sys/socket.h>
#include <unistd.h>

inline void die(const std::string& msg) {
    throw std::runtime_error(msg + ": " + std::strerror(errno));
}

// Send all `len` bytes, looping over partial writes; false on peer-closed/error.
inline bool sendAll(int fd, const char* data, size_t len) {
    size_t sent = 0;
    while (sent < len) {
        ssize_t n = ::send(fd, data + sent, len - sent, 0);
        if (n > 0) { sent += static_cast<size_t>(n); continue; }
        if (n < 0 && errno == EINTR) continue;
        return false;
    }
    return true;
}
```

<br>

#### Skeleton — `echo_server.cpp`

```cpp
// echo_server.cpp
#include "net_util.h"
#include <csignal>
#include <iostream>

int makeListener(uint16_t port) {
    int lfd = ::socket(AF_INET, SOCK_STREAM, 0);
    if (lfd < 0) die("socket");

    int yes = 1;
    if (::setsockopt(lfd, SOL_SOCKET, SO_REUSEADDR, &yes, sizeof yes) < 0)
        die("setsockopt");

    sockaddr_in addr{};
    addr.sin_family      = AF_INET;
    addr.sin_addr.s_addr = htonl(INADDR_ANY);   // 0.0.0.0
    addr.sin_port        = htons(port);

    if (::bind(lfd, reinterpret_cast<sockaddr*>(&addr), sizeof addr) < 0)
        die("bind");
    if (::listen(lfd, 128) < 0)
        die("listen");
    return lfd;
}

// Echo bytes back until the client closes (recv returns 0).
void handleClient(int cfd) {
    char buf[4096];
    for (;;) {
        ssize_t n = ::recv(cfd, buf, sizeof buf, 0);
        if (n > 0) {
            if (!sendAll(cfd, buf, static_cast<size_t>(n))) {
                std::cerr << "  send failed; dropping client\n";
                break;
            }
            std::cout << "  echoed " << n << " bytes\n";
        } else if (n == 0) {                 // EOF — client closed
            std::cout << "  client closed connection\n";
            break;
        } else {                             // n < 0
            if (errno == EINTR) continue;    // interrupted; retry
            std::cerr << "  recv error: " << std::strerror(errno) << "\n";
            break;
        }
    }
    ::close(cfd);
}

int main(int argc, char** argv) {
    std::signal(SIGPIPE, SIG_IGN);           // portable: don't die on write-to-closed
    uint16_t port = (argc > 1) ? static_cast<uint16_t>(std::stoi(argv[1])) : 8080;

    int lfd = makeListener(port);
    std::cout << "echo server listening on port " << port << "\n";

    for (;;) {
        sockaddr_in cli{};
        socklen_t   clen = sizeof cli;
        int cfd = ::accept(lfd, reinterpret_cast<sockaddr*>(&cli), &clen);
        if (cfd < 0) {
            if (errno == EINTR) continue;
            std::cerr << "accept error: " << std::strerror(errno) << "\n";
            continue;
        }
        char ip[INET_ADDRSTRLEN];
        ::inet_ntop(AF_INET, &cli.sin_addr, ip, sizeof ip);
        std::cout << "accepted client " << ip << ":" << ntohs(cli.sin_port) << "\n";
        handleClient(cfd);                   // one client at a time (blocking)
    }
    ::close(lfd);
    return 0;
}
```

<br>

#### Skeleton — `echo_client.cpp`

```cpp
// echo_client.cpp
#include "net_util.h"
#include <iostream>
#include <vector>

int main(int argc, char** argv) {
    const char* host = (argc > 1) ? argv[1] : "127.0.0.1";
    uint16_t    port = (argc > 2) ? static_cast<uint16_t>(std::stoi(argv[2])) : 8080;

    int fd = ::socket(AF_INET, SOCK_STREAM, 0);
    if (fd < 0) die("socket");

    sockaddr_in srv{};
    srv.sin_family = AF_INET;
    srv.sin_port   = htons(port);
    if (::inet_pton(AF_INET, host, &srv.sin_addr) != 1) die("inet_pton");

    if (::connect(fd, reinterpret_cast<sockaddr*>(&srv), sizeof srv) < 0) die("connect");
    std::cout << "connected to " << host << ":" << port << "\n";

    const std::vector<std::string> msgs = { "hello", "world", "tcp is a byte stream" };
    char buf[4096];
    for (const auto& m : msgs) {
        if (!sendAll(fd, m.data(), m.size())) { std::cerr << "send failed\n"; break; }
        // Read back the echo. NOTE: TCP may coalesce/split — one recv per send
        // is NOT guaranteed. For this toy we read once and print whatever arrived.
        ssize_t n = ::recv(fd, buf, sizeof buf, 0);
        if (n <= 0) { std::cerr << "recv returned " << n << "\n"; break; }
        std::cout << "sent \"" << m << "\" -> echoed \""
                  << std::string(buf, static_cast<size_t>(n)) << "\"\n";
    }

    ::shutdown(fd, SHUT_WR);   // tell server we're done sending (sends FIN)
    ::close(fd);
    return 0;
}
```

<br>

#### Compile and run

```bash
# Build both
g++ -std=c++17 -Wall -Wextra -Wpedantic -o echo_server echo_server.cpp
g++ -std=c++17 -Wall -Wextra -Wpedantic -o echo_client echo_client.cpp

# Terminal 1:
./echo_server 8080

# Terminal 2:
./echo_client 127.0.0.1 8080

# Or poke it with netcat / standard tools:
printf 'hello\n' | nc 127.0.0.1 8080
```

<br>

#### Expected output

```
# server (terminal 1)
echo server listening on port 8080
accepted client 127.0.0.1:54321
  echoed 5 bytes
  echoed 5 bytes
  echoed 20 bytes
  client closed connection

# client (terminal 2)
connected to 127.0.0.1:8080
sent "hello" -> echoed "hello"
sent "world" -> echoed "world"
sent "tcp is a byte stream" -> echoed "tcp is a byte stream"
```

(On a loopback connection messages usually map 1:1, which is *exactly* the trap — the same code over a real network or under load will coalesce/split. Don't let the clean local output fool you into thinking TCP preserves boundaries.)

<br>

#### Bonus Challenges

1. **Demonstrate the byte-stream truth**: make the client `send` three messages back-to-back *without* recv'ing between them, then do one big `recv`. Observe coalescing. Then add a 50 ms sleep between sends and watch splits change.
2. **Length-framing**: prefix each message with a 4-byte big-endian length (`htonl`) and write a `recvExact(fd, buf, n)` that loops until exactly *n* bytes arrive. Now message boundaries are recovered (preview of Day 47).
3. **Concurrent clients via fork/thread**: handle each accepted client in its own thread (`std::thread`) or `fork()` so multiple clients are served simultaneously. Note the one-thread-per-connection cost.
4. **Graceful shutdown**: handle `SIGINT` to close the listening socket and exit the accept loop cleanly. Explain why `accept` returns `EINTR`.
5. **Connect timeout**: make `connect` time out after 3 s (set the socket non-blocking, `connect`, `poll` on `POLLOUT` with a timeout, then check `SO_ERROR`).
6. **TCP_NODELAY**: set `TCP_NODELAY` to disable Nagle's algorithm. Measure round-trip latency of small messages with and without it; explain the Nagle/delayed-ACK interaction.

<br><br>

---
---

# PART 4: DEEP DIVE — COMMON QUESTIONS

---
---

<br>

<h2 style="color: #2980B9;">📘 45.11 Q&A</h2>

<br>

#### Q1: "Why does `accept` return a *new* fd instead of reusing the listening fd?"

The listening fd is a passive endpoint bound to `(local IP, local port)` whose only job is to receive new connections — it never carries application data. Each accepted connection is uniquely identified by the 4-tuple `(local IP, local port, remote IP, remote port)`, and needs its own fd with its own send/recv buffers. So `accept` mints a fresh fd per client while the listener keeps listening. One listener, many connection fds.

<br>

#### Q2: "What does `SO_REUSEADDR` actually do, and why do I need it?"

When your server closes a connection, that socket sits in `TIME_WAIT` for up to ~60 s to absorb stray packets. While any socket on the port is in `TIME_WAIT`, the kernel refuses to `bind` to it again — so restarting your server fails with `EADDRINUSE`. `SO_REUSEADDR` (set before `bind`) permits binding despite `TIME_WAIT` sockets. It's safe and practically mandatory for restartable servers. It is *not* `SO_REUSEPORT`, which lets multiple sockets share a port for load balancing.

<br>

#### Q3: "Why might `recv` return fewer bytes than I asked for? Is that an error?"

Not an error — it's the normal byte-stream behavior. `recv` returns whatever bytes are currently in the receive buffer, up to your requested length. TCP doesn't preserve message boundaries, so a single `recv` gives you "some bytes that have arrived," which may be fewer than a full message or span multiple messages. To read an exact amount you must loop (`recvExact`). To read a whole message you need framing (Day 47).

<br>

#### Q4: "What does `recv` returning 0 mean? It tripped me up."

Zero means **orderly shutdown / EOF**: the peer called `close()` or `shutdown(SHUT_WR)` and sent a FIN — there will never be more data. Action: `close()` your fd and stop reading. The classic bug is treating `0` like "received zero bytes, keep looping," which spins forever. `0` is distinct from `-1` (error) and `>0` (data); all three must be handled separately.

<br>

#### Q5: "What is `SIGPIPE` and why did my server die when the client disconnected?"

If you `send()`/`write()` to a socket whose peer has closed, the kernel delivers `SIGPIPE`, whose default action **terminates your process**. So a client disconnecting at the wrong moment silently kills your server. Fixes: pass `MSG_NOSIGNAL` to `send` (Linux), set the `SO_NOSIGPIPE` socket option (macOS/BSD), or — portably — `signal(SIGPIPE, SIG_IGN)` and check `errno == EPIPE` from `send`'s `-1` return.

<br>

#### Q6: "What is `EINTR` and when do I retry vs. give up?"

`EINTR` means a blocking syscall (`accept`, `recv`, `send`, `connect`) was interrupted by a signal handler before completing. It's not a real failure — the operation simply didn't get to run. The standard pattern is to **retry the call** (`if (errno == EINTR) continue;`). Distinguish it from genuine errors like `ECONNRESET` (peer aborted — give up on that connection) or `EPIPE` (peer closed while you wrote). Only retry `EINTR`.

<br>

#### Q7: "What's the difference between `close()` and `shutdown()`?"

`close(fd)` releases the file descriptor and, when the last reference drops, tears down the connection (sends FIN). `shutdown(fd, how)` half-closes a direction without releasing the fd: `SHUT_WR` sends a FIN (telling the peer "I'm done sending") while you can still `recv` their remaining data; `SHUT_RD` stops reading. A clean client does `shutdown(SHUT_WR)` to signal end-of-request, drains the response, then `close`. This avoids cutting off in-flight data that a bare `close` might discard.

<br>

#### Q8: "Why one thread per connection? What's the limit?"

With blocking I/O, a thread parked in `recv` can't watch any other socket — so each concurrent connection needs its own thread (or process) to block independently. This is simple and fine for dozens to a few hundred connections. It breaks down at thousands: each thread costs a stack (often 1–8 MB), and context-switching among thousands of threads burns CPU (the "C10k problem"). The scalable alternative is non-blocking sockets + a single-threaded event loop with `epoll`/`kqueue` — Day 46.

<br><br>

---

## Reflection Questions

1. List the five server syscalls in order and state what each one does. Why does `accept` return a new fd?
2. What are the SYN queue and accept queue, and what does the `listen` backlog bound?
3. Explain the `TIME_WAIT` / `EADDRINUSE` problem and how `SO_REUSEADDR` solves it.
4. Why is "TCP is a byte stream, not a message stream" the most important fact in this lesson? What does it force you to build?
5. Enumerate the three `recv` return cases and the correct action for each. What's the EOF bug?
6. Why must both `send` and `recv` be wrapped in loops? What is `SIGPIPE` and how do you handle it portably?

---

## Interview Questions

1. "Walk me through the syscalls a TCP server makes, in order, and what each does."
2. "Why does binding to a port sometimes fail right after you restart a server, and how do you fix it?"
3. "TCP is a byte stream. What does that imply for how you read messages off a socket?"
4. "What do the return values of `recv` mean? How do you handle `0`, `-1`, and a short read?"
5. "Your server crashes when a client disconnects mid-write. Diagnose and fix it."
6. "What's the difference between `close` and `shutdown`? When would you use `shutdown(SHUT_WR)`?"
7. "What is the `listen` backlog, and what happens when it overflows?"
8. "Implement a `sendAll` that reliably writes a full buffer over a blocking socket."
9. "When is one-thread-per-connection acceptable, and when does it break down?"
10. "What is `EINTR`, and how does correct socket code handle it?"

---

**Next**: Day 46 — Non-blocking I/O & Event Loop →
