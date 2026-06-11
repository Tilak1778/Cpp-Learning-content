# Day 36: Design a Logger

[← Back to Study Plan](../lld-study-plan.md) | [← Day 34](day34-parallel-algorithms.md)

> **Time**: ~1.5-2 hours
> **Goal**: A logger is the single most-reused infrastructure component in any real system, and it is a favorite LLD interview question because it touches abstraction (the **sink** interface), formatting, severity filtering, and — later — concurrency. Today you build the *single-threaded core*: log **levels** with compile-time and run-time filtering, structured **formatting** (timestamp, level, thread id, message), the **sink** abstraction (console, file, multi-sink fan-out), and **log rotation** by size. Tomorrow (Day 37) you make it async and thread-safe.

---
---

# PART 1: WHY A LOGGER IS A DESIGN PROBLEM, NOT A `printf`

---
---

<br>

<h2 style="color: #2980B9;">📘 36.1 The Core Idea</h2>

Everyone's first logger is one line:

```cpp
std::cout << "something happened\n";
```

That works until you have to answer questions a real system always asks:

> "Only show me warnings and above in production, but everything in debug builds. Write to a file that rotates at 10 MB. Prefix every line with a timestamp, the severity, and which thread emitted it. And let me add a *second* destination — a network sink — without touching any call site."

Each of those requirements is a **separable concern**. A good logger keeps them separate:

```
   Call site                Logger core              Sink(s)
 ┌───────────┐   record   ┌────────────┐  bytes  ┌──────────────┐
 │ LOG_INFO  │ ─────────► │  filter    │ ──────► │ ConsoleSink  │
 │ ("hi %d") │            │  format    │   ┌───► │ FileSink     │
 └───────────┘            └────────────┘   └───► │ NetworkSink  │
                                                  └──────────────┘
```

The three responsibilities — **decide whether to log** (level filter), **turn a record into text** (formatter), and **put bytes somewhere** (sink) — are the three axes you must keep independent. Conflating them is what makes homegrown loggers impossible to evolve.

<br>

#### What we are building today

| Concern | Today (Day 36) | Tomorrow (Day 37) |
|---------|----------------|-------------------|
| Levels + filtering | ✅ | — |
| Formatting (timestamp, level, message) | ✅ | — |
| Sink abstraction (console, file) | ✅ | — |
| Multi-sink fan-out | ✅ | — |
| Log rotation by size | ✅ | — |
| Thread safety | ❌ (single-threaded core) | ✅ |
| Async background flush | ❌ | ✅ |

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 36.2 Log Levels</h2>

A **level** is a severity ordering. The conventional set (from syslog / most frameworks):

```cpp
enum class Level : int {
    Trace = 0,   // ultra-verbose: function entry/exit, loop iterations
    Debug = 1,   // developer diagnostics
    Info  = 2,   // normal operation milestones ("server started on :8080")
    Warn  = 3,   // recoverable anomaly ("retrying connection")
    Error = 4,   // operation failed, but process survives
    Fatal = 5,   // unrecoverable — usually followed by abort/exit
    Off   = 6    // sentinel: disables all logging
};
```

The key property: levels are **totally ordered** by integer value. A logger has a single **threshold**; a record is emitted only if `record.level >= threshold`.

```
threshold = Warn (3)

  Trace(0) Debug(1) Info(2) | Warn(3) Error(4) Fatal(5)
  └──────── dropped ────────┘ └──────── emitted ───────┘
```

Setting `threshold = Off` drops everything; `threshold = Trace` emits everything.

<br>

#### Run-time vs compile-time filtering

There are two filters, and they stack:

| Filter | When evaluated | Cost of a dropped log |
|--------|----------------|-----------------------|
| **Run-time** | `if (level >= m_threshold)` inside `log()` | A branch + the cost of *evaluating the arguments* |
| **Compile-time** | `#if`/`if constexpr` / macro guard | **Zero** — the call site is removed entirely |

The trap is **argument evaluation**. Consider:

```cpp
logger.log(Level::Debug, expensiveDump(graph));   // expensiveDump ALWAYS runs
```

Even when `Debug` is below threshold and the message is dropped, `expensiveDump(graph)` already executed because C++ evaluates arguments before the call. This is why real loggers use **macros** that guard argument evaluation:

```cpp
#define LOG_DEBUG(logger, ...) \
    do { if ((logger).enabled(Level::Debug)) (logger).log(Level::Debug, __VA_ARGS__); } while(0)
```

Now `expensiveDump(graph)` is only evaluated if `Debug` passes the threshold. This is the single most important performance idea in logging: **a disabled log should cost one predictable branch and nothing else.**

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 36.3 The Log Record</h2>

Before formatting, capture the *structured* facts in a small struct. Separating the record from its textual rendering lets different sinks format differently (a JSON sink and a plain-text sink share the same record).

```cpp
#include <chrono>
#include <thread>
#include <string>

struct LogRecord {
    Level                                 level;
    std::string                           message;       // already-formatted user text
    std::chrono::system_clock::time_point time;
    std::thread::id                       threadId;
    const char*                           file;           // __FILE__
    int                                   line;           // __LINE__
};
```

A record is a *value*: cheap to copy or move, no I/O, no locking. (This matters enormously tomorrow when records get pushed onto a queue and consumed on another thread.)

<br><br>

---
---

# PART 2: FORMATTING

---
---

<br>

<h2 style="color: #2980B9;">📘 36.4 The Formatter</h2>

A formatter turns a `LogRecord` into a line of text. The canonical layout:

```
2026-06-11 14:33:07.482 [INFO ] [t#140256] server.cpp:42  Listening on :8080
└──────── timestamp ────┘ └level┘ └thread┘  └─ origin ─┘  └──── message ────┘
```

```cpp
inline const char* levelName(Level l) {
    switch (l) {
        case Level::Trace: return "TRACE";
        case Level::Debug: return "DEBUG";
        case Level::Info:  return "INFO ";   // padded to 5 chars for column alignment
        case Level::Warn:  return "WARN ";
        case Level::Error: return "ERROR";
        case Level::Fatal: return "FATAL";
        default:           return "?????";
    }
}

inline std::string formatTimestamp(std::chrono::system_clock::time_point tp) {
    using namespace std::chrono;
    auto t  = system_clock::to_time_t(tp);
    auto ms = duration_cast<milliseconds>(tp.time_since_epoch()) % 1000;
    std::tm tm{};
#if defined(_WIN32)
    localtime_s(&tm, &t);
#else
    localtime_r(&t, &tm);   // localtime_r is the thread-safe variant
#endif
    char buf[32];
    std::strftime(buf, sizeof(buf), "%Y-%m-%d %H:%M:%S", &tm);
    char out[40];
    std::snprintf(out, sizeof(out), "%s.%03d", buf, static_cast<int>(ms.count()));
    return out;
}

inline std::string formatRecord(const LogRecord& r) {
    std::ostringstream os;
    os << formatTimestamp(r.time)
       << " [" << levelName(r.level) << "]"
       << " [t#" << r.threadId << "] "
       << r.message << '\n';
    return os.str();
}
```

> **Why `localtime_r`, not `localtime`?** `std::localtime` returns a pointer to a shared static `tm` — two threads calling it race. `localtime_r` (POSIX) / `localtime_s` (Windows) write into a caller-supplied buffer. This bites people the moment the logger goes multi-threaded (Day 37). Fixing it now costs nothing.

<br>

#### Formatting the user message

Two style choices for how the *message* itself is built:

| Style | Example | Notes |
|-------|---------|-------|
| **printf-style** | `log(Info, "user %s id=%d", name, id)` | Type-unsafe; needs varargs; familiar |
| **stream-style** | `LOG(Info) << "user " << name << " id=" << id` | Type-safe; temporary object flushes on destruction |
| **`std::format` (C++20)** | `log(Info, "user {} id={}", name, id)` | Type-safe + compile-time checked format string |

We'll keep the *core* taking a pre-built `std::string message` (so formatting style is orthogonal), and demonstrate a `std::format`-style helper in the build exercise.

<br><br>

---
---

# PART 3: THE SINK ABSTRACTION

---
---

<br>

<h2 style="color: #2980B9;">📘 36.5 What a Sink Is</h2>

A **sink** is *the destination of formatted bytes*. This is the most important abstraction in the whole design — it is what lets you add a file, a network endpoint, or a test-capture buffer without touching a single call site.

```cpp
class ILogSink {
public:
    virtual ~ILogSink() = default;
    virtual void write(const LogRecord& record) = 0;   // emit one record
    virtual void flush() {}                              // force buffered bytes out
};
```

> **Design choice: pass the `LogRecord`, not a pre-formatted string.** If you passed a `std::string`, every sink would be forced to share one format. Passing the *record* lets each sink own its formatter (plain text to console, JSON to a log-aggregation service). The default sinks below format internally; a fancier design injects an `IFormatter` into each sink.

<br>

```
                     ┌──────────────────┐
                     │   ILogSink       │  (interface)
                     │  + write(record) │
                     │  + flush()       │
                     └────────┬─────────┘
            ┌─────────────────┼──────────────────┐
            ▼                 ▼                  ▼
    ┌──────────────┐  ┌──────────────┐  ┌──────────────────┐
    │ ConsoleSink  │  │  FileSink    │  │ RotatingFileSink │
    └──────────────┘  └──────────────┘  └──────────────────┘
                                ▲
                                │ (also a sink — fans out)
                        ┌───────┴────────┐
                        │  MultiSink     │
                        │  vector<sinks> │
                        └────────────────┘
```

Note that `MultiSink` is *itself* a sink — it implements `ILogSink` and forwards to a list of child sinks. This is the **Composite** pattern, and it means the logger only ever talks to one `ILogSink*`.

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 36.6 Console Sink</h2>

```cpp
#include <iostream>

class ConsoleSink : public ILogSink {
public:
    void write(const LogRecord& r) override {
        // stderr is unbuffered and conventional for diagnostics;
        // route Warn+ to stderr, Info- to stdout if you want.
        std::ostream& os = (r.level >= Level::Warn) ? std::cerr : std::cout;
        os << formatRecord(r);
    }
    void flush() override {
        std::cout.flush();
        std::cerr.flush();
    }
};
```

Optionally add ANSI color by severity (red for Error, yellow for Warn) — guard it behind an `isatty()` check so colors don't leak into piped/redirected output.

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 36.7 File Sink</h2>

```cpp
#include <fstream>

class FileSink : public ILogSink {
    std::ofstream m_out;
public:
    explicit FileSink(const std::string& path, bool append = true)
        : m_out(path, append ? std::ios::app : std::ios::trunc) {
        if (!m_out) throw std::runtime_error("FileSink: cannot open " + path);
    }
    void write(const LogRecord& r) override { m_out << formatRecord(r); }
    void flush() override { m_out.flush(); }
};
```

> **Buffering vs durability.** `std::ofstream` buffers internally; bytes hit disk only on `flush()` or when the buffer fills. That's fast but means a crash can lose the last few lines. The trade-off:
>
> - **Flush every record** → safe, slow (a syscall per line)
> - **Flush periodically / on level ≥ Error** → fast, slightly lossy
>
> A common policy: buffer everything, but `flush()` immediately on `Error`/`Fatal` (the records you most want to survive a crash).

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 36.8 Multi-Sink (Composite)</h2>

```cpp
#include <vector>
#include <memory>

class MultiSink : public ILogSink {
    std::vector<std::shared_ptr<ILogSink>> m_sinks;
public:
    void add(std::shared_ptr<ILogSink> s) { m_sinks.push_back(std::move(s)); }
    void write(const LogRecord& r) override {
        for (auto& s : m_sinks) s->write(r);
    }
    void flush() override {
        for (auto& s : m_sinks) s->flush();
    }
};
```

Now "log to console *and* a file" is just two `add()` calls — the logger and every call site are untouched.

<br><br>

---
---

# PART 4: LOG ROTATION

---
---

<br>

<h2 style="color: #2980B9;">📘 36.9 Why Rotate</h2>

A long-running service writing to one file produces an unbounded file that fills the disk and is impossible to open in an editor. **Rotation** caps individual file size and bounds total log history.

Two common rotation triggers:

| Trigger | Example | Used when |
|---------|---------|-----------|
| **Size-based** | rotate at 10 MB | High, bursty throughput |
| **Time-based** | rotate at midnight | Predictable daily volume, easy archival (`app-2026-06-11.log`) |

We implement **size-based** today (the more interesting mechanic). The scheme:

```
Before rotation (app.log hits 10 MB):
   app.log        ← current

Rotation step (shift everyone down):
   app.log    → app.1.log
   app.1.log  → app.2.log
   ...
   app.(N-1)  → app.N.log
   app.N.log  → deleted   (oldest dropped — bounds total history)

After rotation:
   app.log        ← fresh, empty
   app.1.log ... app.N.log   (newest..oldest backups)
```

`maxFiles = N` bounds total disk use at roughly `maxBytes * (N + 1)`.

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 36.10 Rotating File Sink</h2>

```cpp
#include <cstdio>     // std::rename, std::remove
#include <filesystem>

class RotatingFileSink : public ILogSink {
    std::string   m_basePath;     // e.g. "app.log"
    std::size_t   m_maxBytes;
    int           m_maxFiles;     // number of backups to keep
    std::ofstream m_out;
    std::size_t   m_written = 0;  // bytes written to current file

public:
    RotatingFileSink(std::string path, std::size_t maxBytes, int maxFiles)
        : m_basePath(std::move(path)), m_maxBytes(maxBytes), m_maxFiles(maxFiles) {
        openCurrent(/*append=*/true);
    }

    void write(const LogRecord& r) override {
        std::string line = formatRecord(r);
        if (m_written + line.size() > m_maxBytes && m_written > 0) {
            rotate();
        }
        m_out << line;
        m_written += line.size();
    }

    void flush() override { m_out.flush(); }

private:
    std::string backupName(int i) const { return m_basePath + "." + std::to_string(i); }

    void openCurrent(bool append) {
        m_out.close();
        m_out.open(m_basePath, append ? std::ios::app : std::ios::trunc);
        if (!m_out) throw std::runtime_error("RotatingFileSink: cannot open " + m_basePath);
        // resume the running byte count from existing file size on append
        std::error_code ec;
        auto sz = std::filesystem::file_size(m_basePath, ec);
        m_written = ec ? 0 : static_cast<std::size_t>(sz);
    }

    void rotate() {
        m_out.close();
        // delete the oldest, then shift each backup down by one
        std::remove(backupName(m_maxFiles).c_str());
        for (int i = m_maxFiles - 1; i >= 1; --i) {
            std::rename(backupName(i).c_str(), backupName(i + 1).c_str());
        }
        std::rename(m_basePath.c_str(), backupName(1).c_str());
        openCurrent(/*append=*/false);   // fresh empty current file
        m_written = 0;
    }
};
```

> **Why `> 0` in the rotation guard?** If a single record is larger than `maxBytes`, you must still write it somewhere — refusing to write or infinite-rotating would lose data. The `m_written > 0` check means "only rotate if the current file is non-empty," so an oversized first record lands in a fresh file rather than triggering an endless rotate loop.

<br><br>

---
---

# PART 5: ASSEMBLING THE LOGGER

---
---

<br>

<h2 style="color: #2980B9;">📘 36.11 The Logger Class</h2>

```cpp
class Logger {
    Level                      m_threshold = Level::Info;
    std::shared_ptr<ILogSink>  m_sink;

public:
    explicit Logger(std::shared_ptr<ILogSink> sink, Level threshold = Level::Info)
        : m_threshold(threshold), m_sink(std::move(sink)) {}

    void setLevel(Level l) { m_threshold = l; }
    bool enabled(Level l) const { return l >= m_threshold; }

    void log(Level l, std::string msg,
             const char* file = "", int line = 0) {
        if (!enabled(l)) return;                       // run-time filter
        LogRecord r{ l, std::move(msg),
                     std::chrono::system_clock::now(),
                     std::this_thread::get_id(), file, line };
        m_sink->write(r);
        if (l >= Level::Error) m_sink->flush();        // durability for severe records
    }
};
```

Note the design keeps the logger *thin*: it filters, builds a record, and hands it to a single `ILogSink`. All the variety (console/file/multi/rotating) lives behind that one pointer.

<br>

#### Macros: the zero-cost-when-disabled front door

```cpp
#define LOG(logger, lvl, msg) \
    do { if ((logger).enabled(lvl)) (logger).log((lvl), (msg), __FILE__, __LINE__); } while (0)

#define LOG_INFO(logger, msg)  LOG((logger), Level::Info,  (msg))
#define LOG_WARN(logger, msg)  LOG((logger), Level::Warn,  (msg))
#define LOG_ERROR(logger, msg) LOG((logger), Level::Error, (msg))
```

The `do { } while(0)` idiom makes the macro a single statement that works correctly after `if (x)` without braces and forces a trailing semicolon.

<br><br>

---
---

# PART 6: BUILD EXERCISE

---
---

<br>

<h2 style="color: #2980B9;">📘 36.12 Exercise: A Leveled, Multi-Sink, Rotating Logger</h2>

Build a single-threaded logger with levels, timestamped formatting, a sink interface, a console sink, a rotating file sink, and a multi-sink fan-out. Verify behavior with asserts and by inspecting rotated files.

<br>

#### Skeleton

```cpp
// logger.h
#pragma once
#include <chrono>
#include <cstdio>
#include <ctime>
#include <filesystem>
#include <fstream>
#include <iostream>
#include <memory>
#include <sstream>
#include <stdexcept>
#include <string>
#include <thread>
#include <vector>

enum class Level : int { Trace=0, Debug=1, Info=2, Warn=3, Error=4, Fatal=5, Off=6 };

inline const char* levelName(Level l) {
    switch (l) {
        case Level::Trace: return "TRACE";
        case Level::Debug: return "DEBUG";
        case Level::Info:  return "INFO ";
        case Level::Warn:  return "WARN ";
        case Level::Error: return "ERROR";
        case Level::Fatal: return "FATAL";
        default:           return "?????";
    }
}

struct LogRecord {
    Level                                 level;
    std::string                           message;
    std::chrono::system_clock::time_point time;
    std::thread::id                       threadId;
};

inline std::string formatTimestamp(std::chrono::system_clock::time_point tp) {
    using namespace std::chrono;
    auto t  = system_clock::to_time_t(tp);
    auto ms = duration_cast<milliseconds>(tp.time_since_epoch()) % 1000;
    std::tm tm{};
#if defined(_WIN32)
    localtime_s(&tm, &t);
#else
    localtime_r(&t, &tm);
#endif
    char buf[32];
    std::strftime(buf, sizeof(buf), "%Y-%m-%d %H:%M:%S", &tm);
    char out[40];
    std::snprintf(out, sizeof(out), "%s.%03d", buf, static_cast<int>(ms.count()));
    return out;
}

inline std::string formatRecord(const LogRecord& r) {
    std::ostringstream os;
    os << formatTimestamp(r.time) << " [" << levelName(r.level) << "] "
       << r.message << '\n';
    return os.str();
}

// ── Sink interface ───────────────────────────────────────
class ILogSink {
public:
    virtual ~ILogSink() = default;
    virtual void write(const LogRecord& record) = 0;
    virtual void flush() {}
};

// ── Console sink ─────────────────────────────────────────
class ConsoleSink : public ILogSink {
public:
    void write(const LogRecord& r) override {
        std::ostream& os = (r.level >= Level::Warn) ? std::cerr : std::cout;
        os << formatRecord(r);
    }
    void flush() override { std::cout.flush(); std::cerr.flush(); }
};

// ── Rotating file sink ───────────────────────────────────
class RotatingFileSink : public ILogSink {
    std::string   m_basePath;
    std::size_t   m_maxBytes;
    int           m_maxFiles;
    std::ofstream m_out;
    std::size_t   m_written = 0;
public:
    RotatingFileSink(std::string path, std::size_t maxBytes, int maxFiles)
        : m_basePath(std::move(path)), m_maxBytes(maxBytes), m_maxFiles(maxFiles) {
        openCurrent(true);
    }
    void write(const LogRecord& r) override {
        std::string line = formatRecord(r);
        if (m_written + line.size() > m_maxBytes && m_written > 0) rotate();
        m_out << line;
        m_written += line.size();
    }
    void flush() override { m_out.flush(); }
private:
    std::string backupName(int i) const { return m_basePath + "." + std::to_string(i); }
    void openCurrent(bool append) {
        m_out.close();
        m_out.open(m_basePath, append ? std::ios::app : std::ios::trunc);
        if (!m_out) throw std::runtime_error("cannot open " + m_basePath);
        std::error_code ec;
        auto sz = std::filesystem::file_size(m_basePath, ec);
        m_written = ec ? 0 : static_cast<std::size_t>(sz);
    }
    void rotate() {
        m_out.close();
        std::remove(backupName(m_maxFiles).c_str());
        for (int i = m_maxFiles - 1; i >= 1; --i)
            std::rename(backupName(i).c_str(), backupName(i + 1).c_str());
        std::rename(m_basePath.c_str(), backupName(1).c_str());
        openCurrent(false);
        m_written = 0;
    }
};

// ── Composite multi-sink ─────────────────────────────────
class MultiSink : public ILogSink {
    std::vector<std::shared_ptr<ILogSink>> m_sinks;
public:
    void add(std::shared_ptr<ILogSink> s) { m_sinks.push_back(std::move(s)); }
    void write(const LogRecord& r) override { for (auto& s : m_sinks) s->write(r); }
    void flush() override { for (auto& s : m_sinks) s->flush(); }
};

// ── Logger ───────────────────────────────────────────────
class Logger {
    Level                     m_threshold = Level::Info;
    std::shared_ptr<ILogSink> m_sink;
public:
    explicit Logger(std::shared_ptr<ILogSink> sink, Level t = Level::Info)
        : m_threshold(t), m_sink(std::move(sink)) {}
    void setLevel(Level l) { m_threshold = l; }
    bool enabled(Level l) const { return l >= m_threshold; }
    void log(Level l, std::string msg) {
        if (!enabled(l)) return;
        LogRecord r{ l, std::move(msg),
                     std::chrono::system_clock::now(),
                     std::this_thread::get_id() };
        m_sink->write(r);
        if (l >= Level::Error) m_sink->flush();
    }
};

#define LOG(logger, lvl, msg) \
    do { if ((logger).enabled(lvl)) (logger).log((lvl), (msg)); } while (0)
```

<br>

#### A test capture sink (for assertable tests)

```cpp
// Counts records and remembers the last message — lets us assert on behavior.
class CaptureSink : public ILogSink {
public:
    int count = 0;
    std::string last;
    void write(const LogRecord& r) override { ++count; last = r.message; }
};
```

<br>

#### Test driver

```cpp
// main.cpp
#include "logger.h"
#include <cassert>

int main() {
    // 1. Level filtering
    {
        auto cap = std::make_shared<CaptureSink>();
        Logger log(cap, Level::Warn);
        LOG(log, Level::Info,  "should be dropped");
        LOG(log, Level::Warn,  "kept 1");
        LOG(log, Level::Error, "kept 2");
        assert(cap->count == 2);
        assert(cap->last == "kept 2");
        std::cout << "[1] level filtering OK\n";
    }

    // 2. Dynamic level change
    {
        auto cap = std::make_shared<CaptureSink>();
        Logger log(cap, Level::Error);
        LOG(log, Level::Warn, "dropped");
        assert(cap->count == 0);
        log.setLevel(Level::Trace);
        LOG(log, Level::Trace, "now kept");
        assert(cap->count == 1);
        std::cout << "[2] dynamic level OK\n";
    }

    // 3. Multi-sink fan-out
    {
        auto a = std::make_shared<CaptureSink>();
        auto b = std::make_shared<CaptureSink>();
        auto multi = std::make_shared<MultiSink>();
        multi->add(a);
        multi->add(b);
        Logger log(multi, Level::Trace);
        LOG(log, Level::Info, "fan out");
        assert(a->count == 1 && b->count == 1);
        std::cout << "[3] multi-sink fan-out OK\n";
    }

    // 4. Rotation: tiny maxBytes forces rotation; check backups exist
    {
        const std::string base = "test_app.log";
        std::remove(base.c_str());
        for (int i = 1; i <= 5; ++i) std::remove((base + "." + std::to_string(i)).c_str());

        auto rot = std::make_shared<RotatingFileSink>(base, /*maxBytes=*/80, /*maxFiles=*/3);
        Logger log(rot, Level::Trace);
        for (int i = 0; i < 20; ++i) LOG(log, Level::Info, "rotation line " + std::to_string(i));
        rot->flush();

        assert(std::filesystem::exists(base));
        assert(std::filesystem::exists(base + ".1"));   // at least one backup created
        std::cout << "[4] rotation OK (inspect "
                  << base << ", " << base << ".1, ...)\n";
    }

    // 5. Console sink (visual)
    {
        auto con = std::make_shared<ConsoleSink>();
        Logger log(con, Level::Trace);
        LOG(log, Level::Info,  "hello from console (stdout)");
        LOG(log, Level::Error, "this goes to stderr");
        std::cout << "[5] console sink OK\n";
    }

    std::cout << "All assertions passed.\n";
    return 0;
}
```

<br>

#### Compile and run

```bash
g++ -std=c++17 -Wall -Wextra -Wpedantic -fsanitize=address \
    -o day36 main.cpp && ./day36
```

<br>

#### Expected output pattern

```
[1] level filtering OK
[2] dynamic level OK
[3] multi-sink fan-out OK
[4] rotation OK (inspect test_app.log, test_app.log.1, ...)
2026-06-11 14:33:07.482 [INFO ] hello from console (stdout)
2026-06-11 14:33:07.482 [ERROR] this goes to stderr
[5] console sink OK
All assertions passed.
```

(The two console lines may interleave because one goes to stdout and one to stderr.)

<br>

#### Bonus Challenges

1. **`std::format` front end** — add a variadic `template<class... Args> void log(Level, std::format_string<Args...>, Args&&...)` that builds the message with `std::format` (C++20). Confirm the format string is checked at compile time.

2. **Per-sink levels** — give each sink its own threshold so the console can show `Info+` while the file captures `Trace+`. Where does the filter move — into the logger or into the sink?

3. **Time-based rotation** — add a `DailyFileSink` that rotates when the calendar day changes, naming files `app-YYYY-MM-DD.log`. What state must it track?

4. **ANSI color console** — colorize the level field (red Error, yellow Warn) but only when `isatty(fileno(stdout))` is true, so redirected output stays clean.

5. **Pluggable formatter** — extract an `IFormatter` interface and inject it into each sink, so a `JsonFormatter` and a `TextFormatter` can coexist over the same records.

6. **Crash-safe Error flush** — verify that `Error`/`Fatal` records survive a simulated crash by flushing immediately, while `Info` records may be lost (kill the process before flush and inspect the file).

<br><br>

---
---

# PART 7: DEEP DIVE — COMMON QUESTIONS

---
---

<br>

<h2 style="color: #2980B9;">📘 36.13 Q&A</h2>

<br>

#### Q1: "Why a sink interface instead of just writing to a file directly?"

The sink interface is the **seam** that lets destinations vary independently of call sites. Without it, "also log to the network" means editing the logger and possibly every emitter. With it, you add a `NetworkSink` class and one `add()` call. It also makes the logger **testable**: a `CaptureSink` lets you assert on what was logged without touching real files or stdout.

<br>

#### Q2: "Why pass a `LogRecord` to the sink instead of a finished string?"

Two reasons. First, different sinks may want different formats (JSON for an aggregator, plain text for a tail-able file) — passing the structured record lets each sink format itself. Second, in the async design (Day 37) the record is pushed onto a queue; keeping it as a small movable value is cheaper and avoids formatting on the hot path. Formatting can be deferred to the consumer thread.

<br>

#### Q3: "What is the cost of a log call that gets filtered out?"

With a *function-call* API, the arguments are evaluated before the call, so an expensive message is built even when dropped. With a *macro* API (`LOG(...)`), the `enabled()` branch short-circuits before evaluating anything, so a disabled log costs one predicted branch. This is why every serious logging library ships macros.

<br>

#### Q4: "How does size-based rotation avoid losing data on the boundary?"

Rotation happens *before* the write that would exceed `maxBytes`, never mid-record. The current file is renamed to `app.1`, backups shift down, the oldest is deleted, and a fresh current file is opened. The `m_written > 0` guard prevents an oversized single record from triggering an infinite rotate loop — an oversized record is written whole into a fresh file.

<br>

#### Q5: "Why route Warn+ to stderr and the rest to stdout?"

It's a Unix convention: stdout is for program *output*, stderr for *diagnostics*. Separating them lets an operator do `./app > out.txt 2> errors.txt` and triage errors without scrolling through info noise. stderr is also typically unbuffered, so error lines appear immediately.

<br>

#### Q6: "Should the logger be a singleton?"

Often yes for convenience (one global `Logger& get()`), but couple it loosely: expose the *singleton* as a thin accessor over an injectable `Logger`, so tests can swap the sink. A pure global singleton with a hardcoded file sink is the hard-to-test anti-pattern. Prefer dependency injection at construction, with an optional global for ergonomics.

<br>

#### Q7: "Plain text vs structured (JSON) logs — which?"

Plain text is human-readable and grep-friendly; structured logs are machine-parseable and queryable in tools like Elasticsearch/Splunk. Modern services often emit JSON to a collector and rely on tooling for human reading. The sink+formatter split lets you support both: same records, different formatters/sinks.

<br>

#### Q8: "When do you flush?"

Flushing is a durability-vs-throughput knob. Flushing every record is safe but slow (a syscall per line). The common compromise: buffer normal records, flush immediately on `Error`/`Fatal` (most worth surviving a crash), and flush periodically (e.g., every 200 ms) — which is exactly what the async logger's background thread does tomorrow.

<br><br>

---

## Reflection Questions

1. Why must a disabled log call cost essentially nothing, and what mechanism achieves that in C++?
2. Why is `localtime` unsafe and `localtime_r` the right choice — even before the logger is multi-threaded?
3. Why is `MultiSink` itself an `ILogSink`? What pattern is that, and what does it buy you?
4. What is the durability/throughput trade-off in `flush()`, and what's a sensible default policy?
5. How does passing a `LogRecord` (vs a formatted string) to sinks pay off in both formatting flexibility and the async design?
6. What guard prevents size-based rotation from infinite-looping on an oversized single record?

---

## Interview Questions

1. "Design a logging library. Walk me through the main abstractions."
2. "How do you make a disabled `LOG_DEBUG(...)` call have near-zero cost?"
3. "Explain the sink abstraction. How would you add a network destination without touching call sites?"
4. "Implement size-based log rotation. How do you bound total disk usage?"
5. "How would you support both plain-text and JSON output from the same log calls?"
6. "Why pass a structured record to sinks rather than a formatted string?"
7. "What's the difference between compile-time and run-time log filtering?"
8. "How do you decide when to flush? What's the trade-off?"
9. "Why route different severities to stdout vs stderr?"
10. "How would you make this logger testable without writing real files?"

---

**Next**: Day 37 — Logger: Async & Thread-Safe →
