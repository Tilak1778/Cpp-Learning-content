# Day 44: Design a KV Store

[← Back to Study Plan](../lld-study-plan.md) | [← Day 43](day43-connection-pool.md)

> **Time**: ~1.5-2 hours
> **Goal**: A persistent key-value store is the foundation under Redis, RocksDB, etcd, and every database storage engine. The hard part isn't the hash map — it's **durability**: surviving a crash without losing acknowledged writes and without corrupting the data file. Learn the in-memory index (hash map), the **write-ahead log (WAL)** and why we log *before* mutating, `fsync` and the durability/performance trade-off, **crash recovery via log replay**, **compaction** (reclaiming space from overwritten/deleted keys), and the high-level difference between **LSM-trees** (write-optimized) and **B-trees** (read/update-optimized). Build a persistent KV store with `get`/`put`/`delete`, an append-only WAL, and a `load()` that replays the log on startup.

---
---

# PART 1: FROM HASH MAP TO DURABLE STORE

---
---

<br>

<h2 style="color: #2980B9;">📘 44.1 The In-Memory Part Is Easy</h2>

A volatile KV store is just a hash map:

```cpp
std::unordered_map<std::string, std::string> data_;
void put(const std::string& k, const std::string& v) { data_[k] = v; }
std::optional<std::string> get(const std::string& k) {
    auto it = data_.find(k);
    return it == data_.end() ? std::nullopt : std::optional(it->second);
}
void del(const std::string& k) { data_.erase(k); }
```

`get`/`put`/`delete` are all average O(1). Done — except the moment the process exits or crashes, **every byte is gone**. The entire engineering problem of a KV store is the gap between "in RAM" and "durable on disk, surviving a crash mid-write."

Three properties we want:

| Property | Meaning |
|----------|---------|
| **Durability** | Once `put` returns success, the value survives a crash. |
| **Atomicity** | A write either fully happens or not at all — never a half-written record. |
| **Recoverability** | After a crash, we can reconstruct the exact last consistent state. |

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 44.2 Why Not Just Rewrite the File Each Put?</h2>

The naive durable approach — serialize the whole map and rewrite the file on every `put` — fails on two counts:

1. **Performance**: O(N) disk writes per `put`. A 1 GB dataset means writing 1 GB to durably store one 10-byte change.
2. **Atomicity**: if the process crashes *during* the rewrite, the file is half old / half new — corrupt and unrecoverable. The OS makes no promise that a large `write()` is atomic.

The insight that powers nearly all storage engines: **don't mutate data in place; append a record describing the change to a log.** Appends are sequential (fast even on spinning disks), small (one record per change), and crash-safe (a torn append at the tail is detectable and discardable).

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 44.3 The Write-Ahead Log (WAL)</h2>

The WAL is an **append-only file** of operations. The rule is in the name: **write to the log *before* applying the change to the in-memory state (and before acknowledging the client).**

```
put("a","1")  →  append "PUT a 1\n" to WAL ─► fsync ─► data_["a"]="1" ─► return OK
del("a")      →  append "DEL a\n"   to WAL ─► fsync ─► data_.erase("a") ─► return OK
```

```
   WAL on disk (append-only, grows forever until compaction):
   ┌──────────────────────────────────────────────────────────┐
   │ PUT user:1 alice │ PUT user:2 bob │ DEL user:1 │ PUT ...   │
   └──────────────────────────────────────────────────────────┘
        offset 0          ...                            tail (append here)
```

Why before? Because the in-memory map is the *fast cache* and the log is the *source of truth*. If we apply the change first and crash before logging, the write is lost — but we already told the client it succeeded. By logging first and fsyncing, we guarantee the acknowledged write is recoverable.

On startup, **replay the log**: read every record in order and re-apply it to the empty map. The final state is identical to the pre-crash state.

```
recovery:  map = {}; for each record in WAL: apply(record);  // PUT → set, DEL → erase
```

This is the same mechanism as a database redo log, ext4's journal, and Raft's log.

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 44.4 The Hybrid: In-Memory Index + On-Disk Log (Bitcask)</h2>

The simplest production-grade design (used by Riak's **Bitcask**) keeps:

- An **append-only data file** on disk holding every record ever written.
- An **in-memory hash map** from key → *offset of its latest record in the file*.

```
On-disk log:                      In-memory index (key → file offset):
  off 0:  PUT a "x"                  "a" → 40   (points at the LATEST "a" record)
  off 14: PUT b "y"                  "b" → 14
  off 28: PUT a "z"  (overwrite)     "c" → (absent — deleted)
  off 40: PUT a "x2"
  off 55: DEL c
```

`get(k)`: look up the offset in the map, seek to it, read one record → one disk seek. `put(k,v)`: append a record, update the map's offset. `delete(k)`: append a tombstone, erase from the map. Old records for `a` (at offsets 0 and 28) are now dead weight — reclaimed later by **compaction**.

This is the design we'll build (simplified: values live in memory too, log records carry both key and value).

<br><br>

---
---

# PART 2: DURABILITY — FSYNC AND THE OS

---
---

<br>

<h2 style="color: #2980B9;">📘 44.5 write() Is Not Durable — You Need fsync()</h2>

A successful `write()` (or `fwrite`) does **not** mean the data is on disk. It means the data is in the OS **page cache** — kernel RAM. The kernel flushes to the physical device lazily (seconds later). If the *machine* loses power before that flush, the data is gone even though `write()` returned success.

```
  application
      │ write()                    ┌──── lost on power failure ────┐
      ▼                            ▼                               │
  ┌────────────┐  copy   ┌──────────────────┐  flush   ┌──────────┴──┐
  │ user buffer│ ──────► │ OS page cache(RAM)│ ───────► │ disk platter│
  └────────────┘         └──────────────────┘  (lazy)  └─────────────┘
                              ▲
                         fsync() forces this flush NOW and waits for it
```

`fsync(fd)` blocks until the kernel has pushed all dirty pages for that file to stable storage and the device confirms. *That* is your durability point.

```cpp
::write(fd_, record.data(), record.size());   // in page cache, NOT durable
::fsync(fd_);                                  // now durable — survives power loss
```

(On macOS, `fsync` historically did *not* flush the drive's own write cache; `fcntl(fd, F_FULLFSYNC)` does. On Linux, `fsync` flushes through to the device on properly behaved hardware.)

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 44.6 The Durability / Throughput Trade-off</h2>

`fsync` is *slow* — milliseconds, because it waits for the physical device. Fsyncing on every single `put` caps you at a few thousand writes/sec on an SSD, far fewer on HDD. So every storage engine offers a knob:

| fsync policy | Durability guarantee | Throughput | Used by |
|--------------|----------------------|------------|---------|
| **Every write** | No acknowledged write ever lost | Lowest | financial ledgers, etcd default |
| **Every N ms (group commit)** | Lose ≤ N ms of writes on power loss | High | Postgres `commit_delay`, Redis `everysec` |
| **Never (OS decides)** | Lose whatever's in page cache (seconds) | Highest | caches, Redis `appendfsync no` |

**Group commit** is the sweet spot: batch many writes, append them all, then a single `fsync` covers the whole batch. One fsync amortized over hundreds of writes. The trade is a bounded window (e.g., 1 second) of writes that could be lost on a hard crash — acceptable for most workloads, unacceptable for a bank.

```
Naive:       put fsync put fsync put fsync put fsync   (4 fsyncs)
Group commit: put put put put ───── fsync               (1 fsync, all 4 durable)
```

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 44.7 Torn Writes and Record Framing</h2>

A crash can interrupt an append mid-record, leaving a **torn write** — a partial record at the tail of the WAL. Recovery must detect and discard it without choking. Two defenses:

1. **Length-prefix each record** (see Day 47 framing): write `[u32 length][payload]`. On replay, if fewer than `length` bytes remain in the file, the record is torn → stop replaying there.
2. **Checksum each record**: write `[crc32][length][payload]`. On replay, recompute the CRC; mismatch → torn or corrupt → stop. This also catches bit-rot, not just truncation.

```
   good record            good record           TORN (crash here)
  ┌────┬─────┬─────────┐ ┌────┬─────┬─────────┐ ┌────┬─────┬───────
  │crc │ len │ payload │ │crc │ len │ payload │ │crc │ len │ payl…   ← only 3 of 8 bytes
  └────┴─────┴─────────┘ └────┴─────┴─────────┘ └────┴─────┴───────
                                                  ▲ replay detects short read → discard, truncate
```

The key principle: **a torn tail record is fine to discard** because we never acknowledged it (the `fsync` for that write hadn't completed when we crashed). What we must never do is silently accept a corrupt record as valid data.

<br><br>

---
---

# PART 3: COMPACTION & STORAGE STRUCTURES

---
---

<br>

<h2 style="color: #2980B9;">📘 44.8 Compaction — Reclaiming Dead Records</h2>

An append-only log grows forever. Overwriting key `a` ten times leaves nine dead records; deleting a key leaves the value *and* the tombstone. Without cleanup the log bloats and recovery slows. **Compaction** rewrites the log keeping only the *latest* record per live key:

```
   Before compaction (8 records, lots of dead weight):
   PUT a 1 │ PUT b 2 │ PUT a 9 │ DEL b │ PUT c 3 │ PUT a 7 │ PUT b 5 │ DEL c

   After compaction (only latest live records survive):
   PUT a 7 │ PUT b 5
   (a's old values gone; c was deleted so dropped; b's tombstone+revive collapsed to PUT b 5)
```

The crash-safe compaction recipe:

1. Write the compacted records to a **new** file (`store.log.compact`).
2. `fsync` the new file.
3. **Atomically `rename()`** it over the old file. POSIX `rename` is atomic — readers see either the old file or the new one, never a mix.
4. Rebuild the in-memory index to point at the new file's offsets.

Doing it as new-file-then-rename means a crash *during* compaction leaves the original log untouched and recoverable. Never compact in place.

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 44.9 LSM-Trees vs B-Trees (Overview)</h2>

The two dominant on-disk structures, optimized for opposite workloads:

| | **LSM-Tree** (RocksDB, Cassandra, LevelDB) | **B-Tree / B+Tree** (Postgres, MySQL InnoDB, SQLite) |
|---|---|---|
| **Write path** | Append to in-memory *memtable* + WAL; flush sorted runs (SSTables) to disk | Find leaf page, update in place (with WAL) |
| **Writes** | Very fast — all sequential appends | Slower — random page reads + in-place updates |
| **Reads** | Slower — may check memtable + several SSTable levels | Fast — one tree traversal, ~log(N) page reads |
| **Space** | Write amplification from compaction; tombstones linger | Lower; fragmentation possible |
| **Best for** | Write-heavy, ingest-heavy (logs, time series, metrics) | Read-heavy, point lookups, range scans, OLTP |

The mental model: an **LSM-tree turns random writes into sequential writes** (write-optimized) by buffering in memory and merging sorted runs in the background (compaction). A **B-tree keeps data sorted in pages and updates in place** (read/update-optimized). Our Bitcask-style build is LSM-*adjacent* (append-only log + in-memory index + compaction) but without the multi-level sorted structure.

```
LSM write flow:                          B+Tree:
  put ─► memtable (sorted, RAM) ─► WAL      root
            │ when full, flush               ├── internal
            ▼                                │    ├── leaf [k,v][k,v]   ◄─ update in place
        SSTable L0 ──┐                       │    └── leaf [k,v][k,v]
        SSTable L0 ──┤ background             └── internal
        SSTable L0 ──┘ compaction ─► L1 ...        └── leaf [k,v]
```

<br><br>

---
---

# PART 4: BUILD EXERCISE

---
---

<br>

<h2 style="color: #2980B9;">📘 44.10 Exercise: A Persistent KV Store with WAL & Recovery</h2>

Build a Bitcask-flavored store: an in-memory `unordered_map`, an append-only WAL with length-prefixed + CRC'd records, `fsync` on write (with a togglable policy), crash recovery by log replay (discarding any torn tail), and a `compact()` that rewrites the log via rename.

> **Portability note**: Uses POSIX `open`/`write`/`fsync`/`rename` (`<fcntl.h>`, `<unistd.h>`). Builds on Linux and macOS. On macOS swap `fsync` → `fcntl(fd, F_FULLFSYNC)` for true device-level durability (noted in code).

<br>

#### Skeleton

```cpp
// kv_store.h
#pragma once
#include <cstdint>
#include <cstring>
#include <fcntl.h>
#include <optional>
#include <stdexcept>
#include <string>
#include <unistd.h>
#include <unordered_map>
#include <vector>

// Record framing on disk:  [u32 crc][u8 op][u32 klen][u32 vlen][key][value]
//   op: 0 = PUT, 1 = DEL (vlen=0)
enum class Op : uint8_t { Put = 0, Del = 1 };

// Tiny CRC32 (table-free, good enough to catch torn/corrupt records).
inline uint32_t crc32(const uint8_t* p, size_t n) {
    uint32_t c = 0xFFFFFFFFu;
    for (size_t i = 0; i < n; ++i) {
        c ^= p[i];
        for (int k = 0; k < 8; ++k)
            c = (c >> 1) ^ (0xEDB88320u & (~(c & 1) + 1)); // branchless mask
    }
    return ~c;
}

class KVStore {
public:
    explicit KVStore(std::string path, bool fsyncEachWrite = true)
        : path_(std::move(path)), fsyncEach_(fsyncEachWrite)
    {
        load();   // replay existing WAL into memory
    }

    ~KVStore() { if (fd_ >= 0) ::close(fd_); }

    KVStore(const KVStore&) = delete;
    KVStore& operator=(const KVStore&) = delete;

    std::optional<std::string> get(const std::string& key) const {
        auto it = data_.find(key);
        if (it == data_.end()) return std::nullopt;
        return it->second;
    }

    void put(const std::string& key, const std::string& value) {
        appendRecord(Op::Put, key, value);   // WAL first...
        data_[key] = value;                   // ...then in-memory state
    }

    void del(const std::string& key) {
        if (data_.find(key) == data_.end()) return;
        appendRecord(Op::Del, key, {});
        data_.erase(key);
    }

    size_t size() const { return data_.size(); }

    // Rewrite the log keeping only live keys; swap in atomically via rename().
    void compact() {
        const std::string tmp = path_ + ".compact";
        int nfd = ::open(tmp.c_str(), O_WRONLY | O_CREAT | O_TRUNC, 0644);
        if (nfd < 0) throw std::runtime_error("compact: open tmp failed");
        for (const auto& [k, v] : data_)
            writeRecordTo(nfd, Op::Put, k, v);
        ::fsync(nfd);
        ::close(nfd);
        if (::rename(tmp.c_str(), path_.c_str()) != 0)   // atomic swap
            throw std::runtime_error("compact: rename failed");
        // reopen the (now compacted) log for further appends
        ::close(fd_);
        fd_ = ::open(path_.c_str(), O_WRONLY | O_APPEND);
        if (fd_ < 0) throw std::runtime_error("compact: reopen failed");
    }

private:
    void load() {
        int rfd = ::open(path_.c_str(), O_RDONLY);
        if (rfd >= 0) {
            std::vector<uint8_t> buf;
            uint8_t chunk[4096];
            ssize_t n;
            while ((n = ::read(rfd, chunk, sizeof chunk)) > 0)
                buf.insert(buf.end(), chunk, chunk + n);
            ::close(rfd);
            replay(buf);
        }
        // open for appending (create if absent)
        fd_ = ::open(path_.c_str(), O_WRONLY | O_CREAT | O_APPEND, 0644);
        if (fd_ < 0) throw std::runtime_error("load: open append failed");
    }

    void replay(const std::vector<uint8_t>& buf) {
        size_t off = 0;
        const size_t hdr = 4 + 1 + 4 + 4;   // crc + op + klen + vlen
        while (off + hdr <= buf.size()) {
            uint32_t crc, klen, vlen; uint8_t op;
            std::memcpy(&crc,  &buf[off],            4);
            std::memcpy(&op,   &buf[off + 4],        1);
            std::memcpy(&klen, &buf[off + 5],        4);
            std::memcpy(&vlen, &buf[off + 9],        4);
            const size_t total = hdr + klen + vlen;
            if (off + total > buf.size()) break;     // TORN tail → stop
            // verify crc over [op..value]
            uint32_t got = crc32(&buf[off + 4], 1 + 4 + 4 + klen + vlen);
            if (got != crc) break;                   // corrupt → stop
            std::string key(reinterpret_cast<const char*>(&buf[off + hdr]), klen);
            if (static_cast<Op>(op) == Op::Put) {
                std::string val(reinterpret_cast<const char*>(&buf[off + hdr + klen]), vlen);
                data_[key] = std::move(val);
            } else {
                data_.erase(key);
            }
            off += total;
        }
    }

    void appendRecord(Op op, const std::string& key, const std::string& value) {
        writeRecordTo(fd_, op, key, value);
        if (fsyncEach_) ::fsync(fd_);   // macOS: fcntl(fd_, F_FULLFSYNC) for true durability
    }

    static void writeRecordTo(int fd, Op op, const std::string& key, const std::string& value) {
        uint32_t klen = static_cast<uint32_t>(key.size());
        uint32_t vlen = static_cast<uint32_t>(value.size());
        std::vector<uint8_t> body;            // [op][klen][vlen][key][value]
        body.push_back(static_cast<uint8_t>(op));
        appendU32(body, klen);
        appendU32(body, vlen);
        body.insert(body.end(), key.begin(), key.end());
        body.insert(body.end(), value.begin(), value.end());
        uint32_t crc = crc32(body.data(), body.size());

        std::vector<uint8_t> rec;             // [crc][body]
        appendU32(rec, crc);
        rec.insert(rec.end(), body.begin(), body.end());

        size_t written = 0;
        while (written < rec.size()) {        // handle partial writes
            ssize_t w = ::write(fd, rec.data() + written, rec.size() - written);
            if (w < 0) throw std::runtime_error("write failed");
            written += static_cast<size_t>(w);
        }
    }

    static void appendU32(std::vector<uint8_t>& v, uint32_t x) {
        v.push_back(x & 0xFF); v.push_back((x >> 8) & 0xFF);
        v.push_back((x >> 16) & 0xFF); v.push_back((x >> 24) & 0xFF);
    }

    std::string path_;
    bool        fsyncEach_;
    int         fd_ = -1;
    std::unordered_map<std::string, std::string> data_;
};
```

<br>

#### Test driver

```cpp
// main.cpp
#include "kv_store.h"
#include <cassert>
#include <cstdio>
#include <iostream>

int main() {
    const std::string path = "/tmp/day44.wal";
    std::remove(path.c_str());

    std::cout << "=== 1. Put / get / delete (in one process) ===\n";
    {
        KVStore kv(path);
        kv.put("user:1", "alice");
        kv.put("user:2", "bob");
        kv.put("user:1", "alice2");          // overwrite
        assert(kv.get("user:1") == "alice2");
        assert(kv.get("user:2") == "bob");
        assert(!kv.get("missing").has_value());
        kv.del("user:2");
        assert(!kv.get("user:2").has_value());
        std::cout << "  live keys: " << kv.size() << " (expect 1)\n";
        assert(kv.size() == 1);
    }   // store closes; WAL persists on disk

    std::cout << "\n=== 2. Recovery: reopen and replay the WAL ===\n";
    {
        KVStore kv(path);                     // load() replays the log
        assert(kv.get("user:1") == "alice2"); // overwrite survived
        assert(!kv.get("user:2").has_value());// delete survived
        std::cout << "  recovered keys: " << kv.size() << " (expect 1)\n";
        assert(kv.size() == 1);
    }

    std::cout << "\n=== 3. Compaction drops dead records ===\n";
    {
        KVStore kv(path);
        for (int i = 0; i < 100; ++i)
            kv.put("hot", "v" + std::to_string(i)); // 100 writes, 99 dead
        assert(kv.get("hot") == "v99");
        kv.compact();                          // rewrite keeping only latest
        assert(kv.get("hot") == "v99");        // still correct after compaction
        std::cout << "  compacted; 'hot' = " << *kv.get("hot") << "\n";
    }

    std::cout << "\n=== 4. Recovery after compaction ===\n";
    {
        KVStore kv(path);
        assert(kv.get("hot") == "v99");
        assert(kv.get("user:1") == "alice2");
        std::cout << "  recovered keys: " << kv.size() << "\n";
    }

    std::remove(path.c_str());
    std::cout << "\nAll assertions passed.\n";
    return 0;
}
```

<br>

#### Compile and run

```bash
g++ -std=c++17 -Wall -Wextra -Wpedantic -fsanitize=address,undefined \
    -o day44 main.cpp && ./day44
```

<br>

#### Expected output pattern

```
=== 1. Put / get / delete (in one process) ===
  live keys: 1 (expect 1)

=== 2. Recovery: reopen and replay the WAL ===
  recovered keys: 1 (expect 1)

=== 3. Compaction drops dead records ===
  compacted; 'hot' = v99

=== 4. Recovery after compaction ===
  recovered keys: 2

All assertions passed.
```

<br>

#### Bonus Challenges

1. **Crash test**: write a record but `kill -9` the process *between* `write()` and `fsync()` (add an env-controlled abort). Reopen and confirm recovery discards the torn record cleanly. Then corrupt a byte mid-file and confirm the CRC check stops replay there.
2. **Group commit**: add a `flush()` and a `fsyncEachWrite=false` mode where writes accumulate and a background thread fsyncs every 100 ms. Benchmark writes/sec for each policy.
3. **Offset index (true Bitcask)**: store `key → file offset` in memory instead of the value, and `get()` seeks+reads the value from disk. Measure memory savings for large values.
4. **Range scan / sorted store**: replace `unordered_map` with `std::map` and add `scan(lo, hi)`. Discuss how this moves you toward a B-tree/LSM read model.
5. **Checkpointing**: periodically snapshot the full map to a `store.snapshot` file and truncate the WAL to records *after* the snapshot — bounding recovery time. (This is what Redis RDB + AOF rewrite does.)
6. **Concurrency**: make it thread-safe (RW lock for the map, a mutex serializing WAL appends). Then explore sharding the keyspace across N stores to reduce lock contention.

<br><br>

---
---

# PART 5: DEEP DIVE — COMMON QUESTIONS

---
---

<br>

<h2 style="color: #2980B9;">📘 44.11 Q&A</h2>

<br>

#### Q1: "Why write to the log *before* updating the in-memory map?"

Because the log is the durable source of truth and the map is a fast volatile cache. If you update the map first and crash before logging, the write is lost — yet you may already have told the client it succeeded. Log-first-then-fsync guarantees that any write you acknowledged is on disk and will be replayed on recovery. The acronym is literal: **W**rite-**A**head **L**og — the log write precedes the state change.

<br>

#### Q2: "If `write()` succeeded, isn't my data safe?"

No. `write()` only copies into the OS page cache (kernel RAM). The kernel flushes to the physical device lazily. A power failure between `write()` and that flush loses the data even though `write()` returned success. Only `fsync()` (or `fcntl(F_FULLFSYNC)` on macOS) forces the flush and waits for the device to confirm. The durability point is the fsync, not the write.

<br>

#### Q3: "fsync on every write is too slow. What are my options?"

Group commit: batch many writes, append them all, then a single `fsync` makes the whole batch durable — one slow fsync amortized over hundreds of writes. You trade a bounded loss window (the un-fsynced batch, e.g., last 100 ms) for throughput. Redis exposes exactly this as `appendfsync always` / `everysec` / `no`; Postgres as `synchronous_commit` and `commit_delay`. Pick the window your durability requirements allow.

<br>

#### Q4: "How does recovery handle a crash that happened mid-write?"

A crash can leave a partial (torn) record at the tail. Two framing defenses: length-prefix each record (a short read at the tail means torn → stop) and CRC each record (a checksum mismatch means torn or corrupt → stop). It's safe to discard a torn *tail* record because its fsync never completed, so we never acknowledged it. The cardinal rule is to never accept a corrupt record as valid — detect and truncate at the first bad record.

<br>

#### Q5: "Why is compaction done as write-new-then-rename instead of in place?"

`rename()` is atomic on POSIX: a reader (or a recovering process) sees either the entire old file or the entire new file, never a half-written mix. Compacting in place could crash mid-rewrite and corrupt your only copy. New-file → fsync → rename means a crash at any point leaves a complete, consistent file — either the original log (recoverable) or the fully compacted one.

<br>

#### Q6: "What's the difference between an LSM-tree and a B-tree, and when do I pick each?"

An LSM-tree buffers writes in a memtable + WAL and flushes sorted runs (SSTables) to disk, merging them via background compaction — this turns random writes into sequential ones, making it **write-optimized** (RocksDB, Cassandra). A B-tree keeps data sorted in pages and updates in place, giving fast **point reads and range scans** — **read/update-optimized** (Postgres, InnoDB, SQLite). Choose LSM for ingest/write-heavy workloads (logs, metrics, time series); choose B-tree for read-heavy OLTP with frequent point lookups and updates.

<br>

#### Q7: "Why a hash map index over an on-disk log (Bitcask) rather than a tree?"

Bitcask's design is dead simple and gives O(1) point lookups with at most one disk seek (the index holds the file offset). The cost: the *entire keyspace* must fit in RAM (values can live on disk), and you can't do efficient range scans (hash maps aren't ordered). It's ideal when you have many keys, point access, and enough RAM for the keys. For range queries or keys-don't-fit-in-RAM, you need a sorted structure (B-tree or LSM).

<br>

#### Q8: "How do I bound recovery time as the log grows?"

Two mechanisms. Compaction shrinks the log by dropping dead records, so replay processes fewer entries. Checkpointing/snapshotting periodically writes the full current state to a snapshot file and truncates the WAL to only records *after* the snapshot — recovery then loads the snapshot and replays just the recent tail. Redis combines both (RDB snapshot + AOF log rewrite). Without these, recovery time grows unbounded with total writes ever made.

<br><br>

---

## Reflection Questions

1. Why does rewriting the whole file on every `put` fail on both performance *and* atomicity grounds? How does an append-only log fix both?
2. Explain the WAL ordering rule. What's the failure mode if you update memory before logging?
3. What exactly does `fsync` guarantee that `write` does not? Where is the data in between?
4. Describe group commit. What durability window are you accepting in exchange for throughput?
5. How does length-prefix + CRC framing let recovery distinguish a torn tail from valid data?
6. Why must compaction use write-new-then-`rename` rather than rewriting in place?

---

## Interview Questions

1. "Design a persistent key-value store. Start from an in-memory map and make it durable."
2. "What is a write-ahead log? Why must the log write happen before the state change?"
3. "A `put` returns success, then the machine loses power. Is the write safe? Walk through write → page cache → fsync."
4. "fsync on every write kills your throughput. How do you fix it without losing durability guarantees the workload needs?"
5. "How does crash recovery handle a write that was interrupted halfway?"
6. "Explain compaction. How do you do it crash-safely?"
7. "Compare LSM-trees and B-trees. Which would you use for a metrics ingestion pipeline, and which for an OLTP database?"
8. "How would you bound the time it takes to recover after a crash as the dataset grows?"
9. "What is a tombstone, and why does deleting a key still write a record?"
10. "How would you make this KV store thread-safe, and how would you scale it across cores?"

---

**Next**: Day 45 — Sockets & TCP Basics →
