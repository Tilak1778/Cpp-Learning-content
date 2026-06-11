# Day 47: Serialization

[← Back to Study Plan](../lld-study-plan.md) | [← Day 46](day46-nonblocking-io-event-loop.md)

> **Time**: ~1.5-2 hours
> **Goal**: TCP gives you an undelimited byte stream (Day 45); to send structured messages over it you must **serialize** them into bytes and recover them on the other side — correctly across machines with different CPUs and compilers. Learn **endianness** and why `htonl`/`ntohl` exist, the **struct packing/alignment trap** that makes `memcpy`-ing a struct over the wire a portability bug, **length-prefixed framing** (the fix for the byte-stream problem), **TLV** (type-length-value) encoding for extensible/optional fields, **varint** (compact integer encoding, overview), and **versioning** so old and new peers interoperate. Build a binary message codec that serializes message structs to a byte buffer and parses them back, with a framed `recvMessage` for TCP.

---
---

# PART 1: WHY YOU CAN'T JUST SEND THE STRUCT

---
---

<br>

<h2 style="color: #2980B9;">📘 47.1 The Tempting, Wrong Approach</h2>

The obvious idea: take your in-memory struct and `memcpy` its bytes onto the socket.

```cpp
struct Message { uint32_t id; uint16_t type; uint64_t timestamp; };
Message m{42, 7, 1700000000};
::send(fd, &m, sizeof m, 0);          // send the raw bytes — WRONG
```

This *appears* to work between two identical processes on one machine and fails the moment the two sides differ in any of three ways:

1. **Endianness** — the byte order of multi-byte integers differs across CPUs.
2. **Struct padding/alignment** — the compiler inserts invisible padding bytes whose layout isn't guaranteed.
3. **Type sizes** — `long`, `int`, `size_t`, enums vary across platforms and ABIs.

Each is a real, subtle, intermittent bug. The fix is an explicit **wire format**: a defined byte layout you write field-by-field and read field-by-field, independent of how either side lays out its memory.

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 47.2 Endianness</h2>

A multi-byte integer can be stored low-byte-first (**little-endian**, x86/ARM) or high-byte-first (**big-endian**). The 32-bit value `0x0A0B0C0D` lives in memory as:

```
   value 0x0A0B0C0D

   little-endian (x86):   0D 0C 0B 0A   ← lowest address holds least-significant byte
   big-endian   (network): 0A 0B 0C 0D  ← lowest address holds most-significant byte
```

If a little-endian sender `memcpy`s the integer and a big-endian receiver reads it, the receiver sees `0x0D0C0B0A` — garbage. The internet standardized on big-endian as **network byte order**, and POSIX gives conversion functions:

```cpp
#include <arpa/inet.h>
uint32_t net = htonl(host32);   // host → network (32-bit)
uint16_t net = htons(host16);   // host → network (16-bit)
uint32_t host = ntohl(net32);   // network → host
uint16_t host = ntohs(net16);   // network → host
```

On a big-endian host these are no-ops; on little-endian they byte-swap. There's no standard 64-bit `htonll`, so you either compose two 32-bit halves or write your own byte-by-byte serializer (which we do below — it's endian-explicit and needs no helpers).

**Rule**: every multi-byte integer that crosses a process boundary must go through a defined byte order — pick one (big-endian is conventional) and convert on the way out and in.

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 47.3 The Struct Packing / Alignment Trap</h2>

The compiler aligns struct members to their natural boundaries (a `uint64_t` to an 8-byte boundary, etc.) by inserting **padding bytes**. Those padding bytes hold *garbage* and their placement is implementation-defined:

```cpp
struct Message {       //  offset (typical x86-64)        sizeof = 16, NOT 14
    uint32_t id;       //  0..3
    uint16_t type;     //  4..5
    // 2 bytes PADDING  //  6..7   ← inserted so timestamp lands on an 8-byte boundary
    uint64_t timestamp;//  8..15
};
```

```
   bytes on the wire if you memcpy this struct:
   [ id:4 ][ type:2 ][ PAD:2 = garbage ][ timestamp:8 ]
                       ▲ leaks uninitialized stack data; layout differs per compiler/arch
```

Three problems: the padding leaks uninitialized memory (a security issue), `sizeof` differs across platforms, and a `#pragma pack(1)` "fix" creates misaligned access (slow or a fault on strict-alignment CPUs). The real fix is to **never serialize the struct's memory layout** — serialize each field explicitly into a tightly-defined byte sequence. Then padding, alignment, and `sizeof` become irrelevant to the wire format.

<br><br>

---
---

# PART 2: FRAMING — RECOVERING MESSAGE BOUNDARIES

---
---

<br>

<h2 style="color: #2980B9;">📘 47.4 The Byte-Stream Problem, Restated</h2>

From Day 45: TCP doesn't preserve message boundaries. If you serialize message A (10 bytes) and message B (8 bytes) and send both, the receiver's `recv` might return all 18 bytes, or 5 then 13, or 18 split however the kernel chose. The serializer produced correct bytes — but the receiver can't tell where A ends and B begins. **Framing** restores the boundaries the stream erased.

The three framing strategies:

| Strategy | How | Pros / Cons |
|----------|-----|-------------|
| **Length prefix** | write `[len][payload]` | Simple, exact, the standard choice. Must read len first. |
| **Delimiter** | terminate with a sentinel (`\n`, `\0`) | Easy for text; breaks if the byte appears in payload (needs escaping). |
| **Fixed size** | every message is N bytes | Trivial but wasteful and inflexible. |

Length-prefix wins for binary protocols and is what we build.

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 47.5 Length-Prefixed Framing</h2>

Prepend each message with its byte length (a fixed-size, big-endian integer). The reader does a two-step read: first the fixed-size length, then exactly that many payload bytes.

```
   on the wire:
   ┌──────────┬───────────────────┬──────────┬───────────────┐
   │ len=10   │  message A (10B)   │ len=8    │ message B (8B)│
   │ (4 bytes)│                    │ (4 bytes)│               │
   └──────────┴───────────────────┴──────────┴───────────────┘
```

```cpp
// recvExact: loop until exactly n bytes are read (handles partial reads / EOF).
bool recvExact(int fd, void* buf, size_t n) {
    auto* p = static_cast<uint8_t*>(buf);
    size_t got = 0;
    while (got < n) {
        ssize_t r = ::recv(fd, p + got, n - got, 0);
        if (r > 0)  { got += static_cast<size_t>(r); continue; }
        if (r == 0) return false;                 // EOF before full message
        if (errno == EINTR) continue;             // retry
        return false;                             // error
    }
    return true;
}

bool recvMessage(int fd, std::vector<uint8_t>& out) {
    uint32_t netlen;
    if (!recvExact(fd, &netlen, 4)) return false; // 1) read the 4-byte length
    uint32_t len = ntohl(netlen);
    if (len > MAX_MESSAGE) return false;          // 2) sanity cap — see §47.9
    out.resize(len);
    return recvExact(fd, out.data(), len);        // 3) read exactly `len` payload bytes
}
```

Two safety musts: **read the length as a defined-endian integer** (here big-endian via `ntohl`), and **bound-check the length** before allocating — an attacker (or a bug) sending `len = 0xFFFFFFFF` would otherwise make you allocate 4 GB.

<br><br>

---
---

# PART 3: ENCODING SCHEMES

---
---

<br>

<h2 style="color: #2980B9;">📘 47.6 Field-by-Field Serialization (Endian-Explicit)</h2>

Instead of `memcpy`-ing structs, write a small `Writer`/`Reader` that appends/reads each field in a fixed byte order. This sidesteps endianness *and* padding in one stroke:

```cpp
struct Writer {
    std::vector<uint8_t> buf;
    void u8 (uint8_t v)  { buf.push_back(v); }
    void u16(uint16_t v) { buf.push_back(v>>8); buf.push_back(v); }                 // big-endian
    void u32(uint32_t v) { for (int s=24; s>=0; s-=8) buf.push_back(v>>s); }
    void u64(uint64_t v) { for (int s=56; s>=0; s-=8) buf.push_back(v>>s); }
    void str(const std::string& s) { u32(s.size()); buf.insert(buf.end(), s.begin(), s.end()); }
};

struct Reader {
    const uint8_t* p; size_t n; size_t off = 0;
    bool need(size_t k) const { return off + k <= n; }
    uint8_t  u8 () { return p[off++]; }
    uint16_t u16() { uint16_t v = (p[off]<<8)|p[off+1]; off+=2; return v; }
    uint32_t u32() { uint32_t v=0; for (int i=0;i<4;i++) v=(v<<8)|p[off++]; return v; }
    uint64_t u64() { uint64_t v=0; for (int i=0;i<8;i++) v=(v<<8)|p[off++]; return v; }
    std::string str() { uint32_t k=u32(); std::string s(reinterpret_cast<const char*>(p+off),k); off+=k; return s; }
};
```

Now serializing a message is explicit and portable: `w.u32(msg.id); w.u16(msg.type); w.u64(msg.timestamp); w.str(msg.name);` — and deserialization reads them back in the same order. The wire layout is exactly the bytes you wrote, no padding, no `sizeof` surprises, byte order fixed.

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 47.7 TLV — Type-Length-Value</h2>

The field-by-field scheme above is **positional**: both sides must agree on the exact field order and presence. **TLV** makes the format self-describing and extensible — each field is a triple `[type][length][value]`:

```
   ┌──────┬────────┬───────────────┐┌──────┬────────┬─────────┐
   │ type │ length │     value      ││ type │ length │  value  │
   │ (1B) │ (2B)   │   (length B)    ││ (1B) │ (2B)   │ ...     │
   └──────┴────────┴───────────────┘└──────┴────────┴─────────┘
```

```cpp
void writeTLV(Writer& w, uint8_t type, const std::vector<uint8_t>& val) {
    w.u8(type);
    w.u16(static_cast<uint16_t>(val.size()));
    w.buf.insert(w.buf.end(), val.begin(), val.end());
}
```

Why TLV is powerful:

- **Optional fields**: just omit the field — the reader iterates whatever fields are present.
- **Forward compatibility**: a reader that doesn't recognize a `type` can **skip it** using its `length` (skip-unknown) and still parse the rest. This is the foundation of extensible protocols.
- **Order independence**: fields can arrive in any order; the reader keys on `type`.

The cost is overhead (3 bytes per field) and looser typing. TLV is the conceptual core of TIFF, BER/DER (X.509 certificates), many network protocols, and — generalized with field tags + wire types — Protocol Buffers.

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 47.8 Varint — Compact Integers (Overview)</h2>

A fixed `u32` always costs 4 bytes, even for the value `5`. **Varint** (variable-length integer, as used by Protocol Buffers) encodes small numbers in fewer bytes: 7 data bits per byte, with the high bit as a "more bytes follow" continuation flag.

```
   value 1        → 0000 0001                     (1 byte)
   value 300      → 1010 1100  0000 0010          (2 bytes; 0xAC 0x02)
                     ▲cont=1     ▲cont=0
   little-endian groups of 7 bits, high bit = "continue"
```

```cpp
void writeVarint(std::vector<uint8_t>& buf, uint64_t v) {
    while (v >= 0x80) { buf.push_back(static_cast<uint8_t>(v) | 0x80); v >>= 7; }
    buf.push_back(static_cast<uint8_t>(v));
}
uint64_t readVarint(const uint8_t* p, size_t& off) {
    uint64_t v = 0; int shift = 0;
    while (true) {
        uint8_t b = p[off++];
        v |= static_cast<uint64_t>(b & 0x7F) << shift;
        if (!(b & 0x80)) break;                    // high bit clear → last byte
        shift += 7;
    }
    return v;
}
```

Varints make protocols where most integers are small (IDs, counts, lengths) dramatically more compact. Caveat: large or negative numbers are *worse* (a negative int as two's-complement varint is 10 bytes — protobuf uses **zigzag** encoding to fix signed values). It's an overview here; the point is "fixed-width is simple, varint trades CPU for space."

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 47.9 Versioning & Defensive Parsing</h2>

Protocols evolve while old and new peers coexist (rolling deploys, third-party clients). Two mechanisms keep them interoperable:

**Version field**: put a version (and ideally a magic number) in a fixed header so the reader knows how to interpret the rest:

```
   [ magic:2 = 0xCAFE ][ version:1 ][ type:1 ][ payload_len:4 ][ payload... ]
```

**Compatibility rules** (the Protobuf discipline, applicable to any TLV scheme):
- **Never reuse or renumber an existing field tag** — old peers will misread it.
- **Only add new fields as optional** with new tags; old readers skip-unknown them.
- **Never change a field's type** in place; add a new field instead.
- Readers must **tolerate unknown fields** (skip via length) and **missing fields** (use defaults).

**Defensive parsing** (mandatory for anything reading off a network — the input is hostile):
- Bound-check every length before allocating/reading (the `len > MAX_MESSAGE` cap).
- Verify `off + size <= n` before each read (`Reader::need`) — never read past the buffer.
- Reject unknown `magic`/`version` early.
- Treat a truncated or malformed message as "drop the connection," not "trust partial data."

A serializer is easy; a *robust deserializer of untrusted bytes* is where the real engineering (and the security bugs) live.

<br><br>

---
---

# PART 4: BUILD EXERCISE

---
---

<br>

<h2 style="color: #2980B9;">📘 47.10 Exercise: A Binary Message Codec</h2>

Build a codec that serializes a `Message` struct to a length-prefixed, big-endian byte buffer and parses it back, with bounds-checked deserialization. Include a framed `recvMessage`/`sendMessage` pair for TCP. The codec is self-contained and testable in one process (no socket needed to verify correctness).

> **Portability note**: The codec is pure C++17 (`<vector>`, `<cstdint>`, `<string>`) and endian-explicit, so it produces identical bytes on x86, ARM, Linux, and macOS. The optional `sendMessage`/`recvMessage` use POSIX sockets.

<br>

#### Skeleton — `codec.h`

```cpp
// codec.h
#pragma once
#include <cstdint>
#include <stdexcept>
#include <string>
#include <vector>

constexpr uint16_t MAGIC       = 0xCAFE;
constexpr uint8_t  VERSION     = 1;
constexpr uint32_t MAX_MESSAGE = 1u << 20;   // 1 MiB cap — defensive

enum class MsgType : uint8_t { Ping = 1, Chat = 2, Bye = 3 };

struct Message {
    MsgType     type = MsgType::Ping;
    uint32_t    id   = 0;
    uint64_t    ts   = 0;
    std::string text;                        // optional payload
};

// ── Big-endian, padding-free writer ──
struct Writer {
    std::vector<uint8_t> buf;
    void u8 (uint8_t v)  { buf.push_back(v); }
    void u16(uint16_t v) { buf.push_back(v >> 8); buf.push_back(v & 0xFF); }
    void u32(uint32_t v) { for (int s = 24; s >= 0; s -= 8) buf.push_back((v >> s) & 0xFF); }
    void u64(uint64_t v) { for (int s = 56; s >= 0; s -= 8) buf.push_back((v >> s) & 0xFF); }
    void str(const std::string& s) {
        u32(static_cast<uint32_t>(s.size()));
        buf.insert(buf.end(), s.begin(), s.end());
    }
};

// ── Bounds-checked reader (throws on truncation) ──
struct Reader {
    const uint8_t* p;
    size_t         n;
    size_t         off = 0;

    void need(size_t k) const {
        if (off + k > n) throw std::runtime_error("decode: truncated message");
    }
    uint8_t  u8 () { need(1); return p[off++]; }
    uint16_t u16() { need(2); uint16_t v = (uint16_t(p[off]) << 8) | p[off+1]; off += 2; return v; }
    uint32_t u32() { need(4); uint32_t v = 0; for (int i = 0; i < 4; ++i) v = (v << 8) | p[off++]; return v; }
    uint64_t u64() { need(8); uint64_t v = 0; for (int i = 0; i < 8; ++i) v = (v << 8) | p[off++]; return v; }
    std::string str() {
        uint32_t k = u32();
        need(k);
        std::string s(reinterpret_cast<const char*>(p + off), k);
        off += k;
        return s;
    }
};

// ── Encode a Message into a framed buffer: [u32 frame-len][payload] ──
inline std::vector<uint8_t> encode(const Message& m) {
    Writer w;                                // payload: magic|ver|type|id|ts|text
    w.u16(MAGIC);
    w.u8(VERSION);
    w.u8(static_cast<uint8_t>(m.type));
    w.u32(m.id);
    w.u64(m.ts);
    w.str(m.text);

    Writer frame;                            // prepend the 4-byte length
    frame.u32(static_cast<uint32_t>(w.buf.size()));
    frame.buf.insert(frame.buf.end(), w.buf.begin(), w.buf.end());
    return std::move(frame.buf);
}

// ── Decode a payload (length already stripped) into a Message ──
inline Message decode(const uint8_t* data, size_t len) {
    Reader r{data, len};
    uint16_t magic = r.u16();
    if (magic != MAGIC)          throw std::runtime_error("decode: bad magic");
    uint8_t ver = r.u8();
    if (ver != VERSION)          throw std::runtime_error("decode: bad version");
    Message m;
    m.type = static_cast<MsgType>(r.u8());
    m.id   = r.u32();
    m.ts   = r.u64();
    m.text = r.str();
    // (forward-compat: extra trailing bytes from a newer peer are ignored)
    return m;
}
```

<br>

#### Optional — framed send/recv over a socket (`net_codec.h`)

```cpp
// net_codec.h  (POSIX sockets)
#pragma once
#include "codec.h"
#include <arpa/inet.h>
#include <cerrno>
#include <sys/socket.h>
#include <unistd.h>

inline bool sendAll(int fd, const uint8_t* p, size_t n) {
    size_t s = 0;
    while (s < n) {
        ssize_t w = ::send(fd, p + s, n - s, 0);
        if (w > 0) { s += static_cast<size_t>(w); continue; }
        if (w < 0 && errno == EINTR) continue;
        return false;
    }
    return true;
}
inline bool recvExact(int fd, uint8_t* p, size_t n) {
    size_t g = 0;
    while (g < n) {
        ssize_t r = ::recv(fd, p + g, n - g, 0);
        if (r > 0) { g += static_cast<size_t>(r); continue; }
        if (r == 0) return false;                 // EOF
        if (errno == EINTR) continue;
        return false;
    }
    return true;
}

inline bool sendMessage(int fd, const Message& m) {
    auto bytes = encode(m);
    return sendAll(fd, bytes.data(), bytes.size());
}
inline bool recvMessage(int fd, Message& out) {
    uint8_t lenbuf[4];
    if (!recvExact(fd, lenbuf, 4)) return false;
    uint32_t len = (uint32_t(lenbuf[0])<<24)|(lenbuf[1]<<16)|(lenbuf[2]<<8)|lenbuf[3];
    if (len == 0 || len > MAX_MESSAGE) return false;   // defensive bound-check
    std::vector<uint8_t> payload(len);
    if (!recvExact(fd, payload.data(), len)) return false;
    out = decode(payload.data(), payload.size());
    return true;
}
```

<br>

#### Test driver

```cpp
// main.cpp
#include "codec.h"
#include <cassert>
#include <cstdio>
#include <iostream>

int main() {
    std::cout << "=== 1. Round-trip a message ===\n";
    Message m{ MsgType::Chat, 42, 1700000000ULL, "hello, wire format" };
    auto bytes = encode(m);
    std::cout << "  encoded " << bytes.size() << " bytes (4 frame + payload)\n";

    // strip the 4-byte frame length, decode the payload
    uint32_t flen = (uint32_t(bytes[0])<<24)|(bytes[1]<<16)|(bytes[2]<<8)|bytes[3];
    assert(flen == bytes.size() - 4);
    Message back = decode(bytes.data() + 4, flen);
    assert(back.type == m.type && back.id == m.id && back.ts == m.ts && back.text == m.text);
    std::cout << "  round-trip ok: id=" << back.id << " text=\"" << back.text << "\"\n";

    std::cout << "\n=== 2. Endianness is explicit (id=0x01020304) ===\n";
    Message e{ MsgType::Ping, 0x01020304u, 0, "" };
    auto eb = encode(e);
    // payload layout after 4-byte frame len: magic(2) ver(1) type(1) id(4)...
    // id big-endian bytes should be 01 02 03 04 regardless of host CPU.
    size_t idOff = 4 /*frame*/ + 2 /*magic*/ + 1 /*ver*/ + 1 /*type*/;
    assert(eb[idOff]==0x01 && eb[idOff+1]==0x02 && eb[idOff+2]==0x03 && eb[idOff+3]==0x04);
    std::cout << "  id serialized big-endian: 01 02 03 04 (host-independent)\n";

    std::cout << "\n=== 3. Defensive decode rejects garbage ===\n";
    try { decode(bytes.data(), 3); assert(false); }            // truncated
    catch (const std::exception& ex) { std::cout << "  truncated -> " << ex.what() << "\n"; }

    std::vector<uint8_t> badmagic = { 0xDE,0xAD,1, 1,0,0,0,0, 0,0,0,0,0,0,0,0, 0,0,0,0 };
    try { decode(badmagic.data(), badmagic.size()); assert(false); }
    catch (const std::exception& ex) { std::cout << "  bad magic -> " << ex.what() << "\n"; }

    std::cout << "\n=== 4. Empty-text message round-trips ===\n";
    Message p{ MsgType::Bye, 7, 0, "" };
    auto pb = encode(p);
    uint32_t pl = (uint32_t(pb[0])<<24)|(pb[1]<<16)|(pb[2]<<8)|pb[3];
    Message pr = decode(pb.data() + 4, pl);
    assert(pr.type == MsgType::Bye && pr.text.empty());
    std::cout << "  ok\n";

    std::cout << "\nAll assertions passed.\n";
    return 0;
}
```

<br>

#### Compile and run

```bash
g++ -std=c++17 -Wall -Wextra -Wpedantic -fsanitize=address,undefined \
    -o day47 main.cpp && ./day47
```

<br>

#### Expected output

```
=== 1. Round-trip a message ===
  encoded 40 bytes (4 frame + payload)
  round-trip ok: id=42 text="hello, wire format"

=== 2. Endianness is explicit (id=0x01020304) ===
  id serialized big-endian: 01 02 03 04 (host-independent)

=== 3. Defensive decode rejects garbage ===
  truncated -> decode: truncated message
  bad magic -> decode: bad magic

=== 4. Empty-text message round-trips ===
  ok

All assertions passed.
```

<br>

#### Bonus Challenges

1. **TLV fields**: extend the payload with a TLV section for optional fields (e.g., a `priority` and a `sender` only sometimes present). Implement skip-unknown so a reader ignores TLV types it doesn't recognize.
2. **Varint lengths**: replace the fixed `u32` string/frame lengths with varints (§47.8). Measure the byte savings for many small messages, and confirm round-trips still hold.
3. **Versioning**: bump to `VERSION = 2` by appending a new field. Show a v1 decoder reading a v2 message (trailing bytes ignored = forward compatible) and a v2 decoder reading a v1 message (missing field → default = backward compatible).
4. **Wire it to TCP**: use `net_codec.h` with the Day 45 echo server. Send a `Message`, have the server decode + re-encode + echo, and verify the framed `recvMessage` reassembles correctly even when you force tiny `recv` chunks.
5. **Fuzz the decoder**: feed `decode()` thousands of random byte buffers under ASan/UBSan. It must never crash, read OOB, or allocate unbounded memory — only throw. Fix any finding.
6. **Compare to a library**: serialize the same `Message` with Protocol Buffers (or FlatBuffers/Cap'n Proto) and compare wire size, encode/decode speed, and schema-evolution ergonomics against your hand-rolled codec.

<br><br>

---
---

# PART 5: DEEP DIVE — COMMON QUESTIONS

---
---

<br>

<h2 style="color: #2980B9;">📘 47.11 Q&A</h2>

<br>

#### Q1: "Why can't I just `memcpy` a struct onto the socket?"

Three reasons it breaks across machines: (1) **endianness** — the receiver may interpret multi-byte integers in the opposite byte order; (2) **padding** — the compiler inserts alignment bytes whose layout isn't portable, and which leak uninitialized memory; (3) **type sizes** — `long`/`size_t`/enum widths vary by platform/ABI. It happens to work between two identical builds on one machine, which is exactly why the bug ships and then fails in production across heterogeneous hosts. Serialize field-by-field into a defined byte layout instead.

<br>

#### Q2: "What is network byte order and when do I convert?"

Network byte order is big-endian, the convention for integers on the wire. Convert every multi-byte integer on the way *out* (host → network, `htons`/`htonl`) and on the way *in* (network → host, `ntohs`/`ntohl`). On a big-endian host these are no-ops; on little-endian (x86/ARM) they byte-swap. There's no standard 64-bit helper, so for 64-bit values write the bytes explicitly (as the codec does) or compose two 32-bit halves.

<br>

#### Q3: "Why do I need framing if I already serialized correctly?"

Because TCP is a byte stream with no message boundaries (Day 45). Correct serialization gives you the right *bytes*; framing tells the receiver where one message ends and the next begins. Without it, two back-to-back messages arrive as one undelimited blob (or split arbitrarily) and the decoder can't know how many bytes to consume. Length-prefixing each message (read the length, then read exactly that many bytes) is the standard fix.

<br>

#### Q4: "What's the advantage of TLV over positional encoding?"

Positional encoding requires both sides to agree on the exact field order and presence — adding or removing a field breaks compatibility. TLV (`type-length-value`) makes each field self-describing: fields can be optional, arrive in any order, and — crucially — a reader can **skip a field whose type it doesn't recognize** using its length. That skip-unknown property is what lets a protocol evolve while old and new peers interoperate. The cost is a few bytes of overhead per field.

<br>

#### Q5: "What is a varint and when is it worth it?"

A varint encodes integers in a variable number of bytes — 7 bits of value per byte, with the high bit as a continuation flag — so small numbers (the common case for IDs, counts, lengths) take 1–2 bytes instead of a fixed 4 or 8. It's worth it when most of your integers are small and bandwidth/size matters (Protobuf uses it pervasively). The downside: large values take *more* than fixed width, and naive signed encoding is terrible (use zigzag), plus a small CPU cost to encode/decode.

<br>

#### Q6: "How do I evolve a binary protocol without breaking old clients?"

Carry a version (and magic) in a fixed header. Then follow tag-stable rules: never reuse or renumber existing field tags, only add new fields as optional with new tags, never change a field's type in place. Readers must tolerate unknown fields (skip via TLV length) and missing fields (apply defaults). This makes new→old (old reader skips the new field) and old→new (new reader defaults the missing field) both work — exactly the Protobuf compatibility model.

<br>

#### Q7: "What's the most important thing about the deserializer?"

That it treats input as **hostile**. Bound-check every length before allocating (cap message size), verify there are enough bytes left before each read (no out-of-bounds), reject unknown magic/version early, and drop the connection on any malformed message rather than trusting partial data. A serializer is trivial; a deserializer of untrusted network bytes is where buffer overreads, OOM-via-huge-length, and integer-overflow bugs live. Fuzz it.

<br>

#### Q8: "When should I hand-roll a codec vs use Protobuf / FlatBuffers / Cap'n Proto?"

Hand-roll for tiny, fixed, performance-critical protocols where you control both ends and want zero dependencies (e.g., an internal RPC frame, an embedded device). Use a library when you need schema evolution, cross-language clients, generated code, and battle-tested edge-case handling — Protobuf for compact wire + ubiquity, FlatBuffers/Cap'n Proto for zero-copy access without a parse step. The hand-rolled version is great for *learning* exactly what those libraries do under the hood (which is precisely this exercise).

<br><br>

---

## Reflection Questions

1. Name the three reasons `memcpy`-ing a struct over the wire is a portability bug, and how field-by-field serialization fixes all three.
2. What is network byte order, and at which two points must you convert? When is the conversion a no-op?
3. Why does correct serialization still require framing? Compare length-prefix, delimiter, and fixed-size framing.
4. What does TLV's skip-unknown property buy you, and why is it the basis of extensible protocols?
5. Sketch how a varint encodes the value 300. When does varint cost *more* than fixed width?
6. List four rules for evolving a binary protocol without breaking old peers, and four defensive-parsing musts.

---

## Interview Questions

1. "How would you send a struct over a socket between a big-endian and a little-endian machine?"
2. "Why can't you just `memcpy` a struct to the wire? What does struct padding have to do with it?"
3. "What is `htonl`/`ntohl` and when do you call them? What about 64-bit values?"
4. "TCP delivered me a blob of two messages stuck together. How do you recover the boundaries?"
5. "Design a length-prefixed framing layer with a `recvMessage`. What two safety checks are mandatory?"
6. "Explain TLV encoding and how it enables optional fields and protocol evolution."
7. "What is a varint? When would you use it, and what's its weakness for signed numbers?"
8. "How do you version a binary protocol so old and new clients interoperate?"
9. "Your decoder reads untrusted bytes off the network. What must it defend against?"
10. "When would you hand-roll a codec vs reach for Protobuf or Cap'n Proto?"

---

**Next**: Day 48 — Shared Memory IPC →
