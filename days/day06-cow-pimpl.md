# Day 6: Copy-on-Write (COW) & Pimpl Idiom

[← Back to Study Plan](../lld-study-plan.md) | [← Day 5](day05-raii-scope-guard.md)

> **Time**: ~3-4 hours (Saturday — extra time for two meaty topics)
> **Goal**: Understand two advanced C++ idioms — Copy-on-Write for deferred deep copies, and Pimpl for ABI stability and compilation firewalls. Implement both from scratch.

---
---

# PART 1: COPY-ON-WRITE (COW)

---
---

<br>

<h2 style="color: #2980B9;">📘 6.1 The Problem COW Solves</h2>

Remember your `String` class from Day 1? Every time you copy it, you do a full **deep copy** — allocate new memory, copy all bytes:

```cpp
String a("a very very very long string with lots of data");
String b = a;   // DEEP COPY — allocates again, copies every byte
String c = a;   // DEEP COPY — allocates AGAIN

// a, b, c all have independent copies of the same data
// but b and c never get modified — we copied for nothing!
```

This is wasteful when the copies are only read, never modified. What if we could **share** the data until someone actually writes?

```
Without COW:                          With COW:
  a.data_ → "long string..."           a.data_ ──┐
  b.data_ → "long string..."  (copy)              ├──► "long string..."  (shared!)
  c.data_ → "long string..."  (copy)   b.data_ ──┘
                                        c.data_ ──┘
  3 allocations                         1 allocation
```

**Copy-on-Write** = share the data on copy, only make a real (deep) copy when someone tries to **write**.

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 6.2 How COW Works — Step by Step</h2>

<br>

#### Step 1: Share on copy

When you copy a COW string, don't copy the data. Instead, both strings point to the **same buffer** and increment a **reference count**:

```
CowString a("Hello");

Memory:
  a ──► [ refcount: 1 | data: "Hello" ]

CowString b = a;   // copy — just increment refcount, share the buffer

Memory:
  a ──┐
      ├──► [ refcount: 2 | data: "Hello" ]
  b ──┘
```

This is O(1) — no allocation, no byte copying. Just bump a counter and copy a pointer.

<br>

#### Step 2: Copy on write (detach)

When someone tries to **modify** one of the copies, check the refcount:

- If `refcount == 1` → you're the only owner → modify in place (no copy needed)
- If `refcount > 1` → others are sharing → make your own copy first, then modify

```
b[0] = 'J';   // b wants to modify

Step 1: refcount is 2 → need to detach
  - Allocate new buffer
  - Copy "Hello" into it
  - Decrement old buffer's refcount (2 → 1)
  - b now points to new buffer with refcount 1

Memory after detach:
  a ──► [ refcount: 1 | data: "Hello" ]
  b ──► [ refcount: 1 | data: "Hello" ]   (independent copy)

Step 2: Now safe to modify
  b ──► [ refcount: 1 | data: "Jello" ]
```

<br>

#### Step 3: Destruction

When a COW string is destroyed, decrement the refcount:
- If it drops to 0 → free the buffer
- Otherwise → do nothing (other copies still using it)

```
// b goes out of scope
  b's refcount-- → 0 → free b's buffer

// a still alive
  a ──► [ refcount: 1 | data: "Hello" ]
```

This is exactly like `SharedPtr`! In fact, COW is just **shared_ptr semantics applied to the internal data buffer**, with the added rule: detach before mutating.

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 6.3 The Shared Data Block</h2>

We need a heap-allocated block that stores both the data and the reference count:

```
┌─────────────────────────────────────────────┐
│  SharedData                                 │
│  ┌──────────────────┬──────────────────┐    │
│  │  refcount: 2     │  data: "Hello\0" │    │
│  └──────────────────┴──────────────────┘    │
└─────────────────────────────────────────────┘
         ▲                    ▲
         │                    │
    CowString a          CowString b
    (both point to same SharedData)
```

In code:

```cpp
struct SharedData {
    size_t refcount;
    size_t size;
    char data[];   // flexible array member — data follows the struct in memory
};
```

Or simpler (for learning):

```cpp
struct SharedData {
    size_t refcount;
    size_t size;
    char* data;    // separate allocation for simplicity
};
```

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 6.4 The Detach Operation — Heart of COW</h2>

Every mutating operation must call `detach()` first:

```cpp
void detach() {
    if (shared_->refcount == 1) return;  // sole owner — nothing to do

    // Multiple owners — make our own copy
    SharedData* new_data = allocate_and_copy(shared_->data, shared_->size);
    new_data->refcount = 1;

    shared_->refcount--;    // leave the shared group
    shared_ = new_data;     // point to our private copy
}
```

After `detach()`, we're guaranteed to be the **sole owner** of `shared_`, so mutation is safe.

<br>

#### Which operations need `detach()`?

| Operation | Needs detach? | Why |
|-----------|:---:|-----|
| `operator[]` (non-const) | **Yes** | Caller might write through the reference |
| `append()`, `insert()`, `erase()` | **Yes** | Modifying content |
| `c_str()` (const) | No | Read-only |
| `operator[]` (const) | No | Read-only |
| `size()`, `empty()` | No | Read-only |
| Copy constructor | No | Just sharing — increment refcount |
| `operator==` | No | Read-only comparison |

<br>

#### The sneaky problem with non-const `operator[]`

```cpp
char& operator[](size_t i) {
    detach();          // must detach because we return a non-const reference
    return shared_->data[i];
}
```

Even if the caller only **reads** through the returned reference (`char c = s[0];`), we have no way to know — we returned a `char&` which **could** be written to. So we must conservatively detach.

This is why COW was removed from `std::string` in C++11 — `operator[]` was detaching too aggressively, killing performance.

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 6.5 COW and Thread Safety</h2>

COW has a subtle threading problem:

```cpp
CowString a("Hello");
CowString b = a;   // a and b share the same SharedData

// Thread 1: reads a (no detach — just reads)
// Thread 2: reads b (no detach — just reads)
// Both threads access shared_->data — fine, both read-only

// But what if Thread 1 reads a[0] while Thread 2 calls b.append("!")?
// Thread 2 does detach() → decrements refcount
// Thread 1 simultaneously reads the refcount to decide if it needs to detach
// → DATA RACE on refcount!
```

The fix: make `refcount` atomic (like your Day 4 `SharedPtr`). But this adds overhead to **every** COW string operation.

**This is the main reason `std::string` stopped using COW in C++11.** The C++11 standard requires that concurrent reads of the same `std::string` are safe without locks, but with COW, even reads can modify the refcount (through detach checks). Making all refcount operations atomic was deemed too expensive.

For your learning implementation, using a non-atomic refcount is fine (single-threaded).

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 6.6 When IS COW Still Useful?</h2>

Even though `std::string` no longer uses it, COW is valuable in other contexts:

| Use Case | Why COW Helps |
|----------|--------------|
| **Immutable data structures** (functional programming) | Copies are free; only new versions allocate |
| **Undo/redo systems** | Store "snapshots" cheaply; only modified pages are duplicated |
| **Operating system fork()** | Process memory is COW — child shares parent's pages until one writes |
| **Database MVCC** | Readers see old version while writer creates new version |
| **Qt's implicit sharing** | `QString`, `QVector`, etc. all use COW for cheap copies |
| **Large config/document objects** | Pass copies around; most readers never modify |

The pattern works best when **copies far outnumber modifications**.

<br><br>

---
---

# PART 2: PIMPL IDIOM

---
---

<br>

<h2 style="color: #2980B9;">📘 6.7 The Problem Pimpl Solves</h2>

Consider a header file:

```cpp
// HttpClient.h
#include <openssl/ssl.h>     // huge header
#include <curl/curl.h>        // another huge header
#include <map>
#include <string>

class HttpClient {
public:
    void get(const std::string& url);
    void post(const std::string& url, const std::string& body);

private:
    SSL_CTX* ssl_ctx_;                         // from OpenSSL
    CURL* curl_handle_;                        // from libcurl
    std::map<std::string, std::string> headers_;
    int timeout_ms_;
    bool follow_redirects_;
    // ... 20 more private members ...
};
```

**Problems:**

1. **Compilation speed**: Anyone who `#include "HttpClient.h"` also pulls in `<openssl/ssl.h>` and `<curl/curl.h>`. If you have 200 source files that use `HttpClient`, all 200 need to parse OpenSSL and curl headers. Change one private member → **200 files recompile**.

2. **ABI stability**: If you add, remove, or reorder a private member, the **size** of `HttpClient` changes. Any binary compiled against the old header is now incompatible → need to recompile all downstream code. This is a nightmare for shared libraries (.so/.dylib).

3. **Coupling**: Users of `HttpClient` don't need to know about OpenSSL or curl. These are implementation details leaking through the header.

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 6.8 What Is Pimpl?</h2>

**Pimpl** = **Pointer to Implementation**. You hide all private members behind an opaque pointer to a forward-declared class:

```cpp
// HttpClient.h — PUBLIC header (what users include)
#include <string>
#include <memory>

class HttpClient {
public:
    HttpClient();
    ~HttpClient();

    HttpClient(HttpClient&&) noexcept;
    HttpClient& operator=(HttpClient&&) noexcept;

    void get(const std::string& url);
    void post(const std::string& url, const std::string& body);

private:
    struct Impl;                      // forward declaration — NOT defined here
    std::unique_ptr<Impl> pimpl_;     // opaque pointer
};
```

```cpp
// HttpClient.cpp — PRIVATE implementation (users never see this)
#include "HttpClient.h"
#include <openssl/ssl.h>      // only included in .cpp!
#include <curl/curl.h>        // only included in .cpp!
#include <map>

struct HttpClient::Impl {
    SSL_CTX* ssl_ctx_;
    CURL* curl_handle_;
    std::map<std::string, std::string> headers_;
    int timeout_ms_ = 5000;
    bool follow_redirects_ = true;
};

HttpClient::HttpClient() : pimpl_(std::make_unique<Impl>()) {
    pimpl_->ssl_ctx_ = SSL_CTX_new(TLS_method());
    pimpl_->curl_handle_ = curl_easy_init();
}

HttpClient::~HttpClient() = default;   // unique_ptr handles cleanup

void HttpClient::get(const std::string& url) {
    // use pimpl_->curl_handle_, pimpl_->headers_, etc.
}
```

<br>

#### What changed?

```
BEFORE Pimpl:                              AFTER Pimpl:
  HttpClient.h                               HttpClient.h
  ┌─────────────────────┐                   ┌─────────────────────┐
  │ #include <openssl>   │                   │ struct Impl;        │  ← forward decl only
  │ #include <curl>      │                   │ unique_ptr<Impl>    │  ← 8 bytes always
  │ SSL_CTX* ssl_ctx_    │                   └─────────────────────┘
  │ CURL* curl_handle_   │
  │ map<...> headers_    │                   HttpClient.cpp
  │ int timeout_ms_      │                   ┌─────────────────────┐
  │ bool follow_redir_   │                   │ #include <openssl>   │  ← only here
  └─────────────────────┘                   │ #include <curl>      │
                                            │ struct Impl { ... }  │
  Users include OpenSSL,                    └─────────────────────┘
  curl, and everything.
  Change a member →                          Users only see the pointer.
  EVERYTHING recompiles.                     Change Impl → only .cpp recompiles.
```

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 6.9 Why Does Pimpl Work? — The Compilation Firewall</h2>

When the compiler sees `struct Impl;` (forward declaration), it knows `Impl` exists as a type, but doesn't know its size, members, or layout. You **cannot** do:

```cpp
Impl obj;               // ✗ — compiler doesn't know sizeof(Impl)
obj.member = 5;          // ✗ — compiler doesn't know what members exist
```

But you **can** do:

```cpp
Impl* ptr;               // ✓ — pointer is always 8 bytes regardless of Impl's size
std::unique_ptr<Impl> p; // ✓ — unique_ptr stores a pointer (8 bytes)
```

This is key: **a pointer to an incomplete type is valid.** The compiler knows a pointer is 8 bytes (on 64-bit) without knowing what it points to.

So `sizeof(HttpClient)` is always the same: just the `unique_ptr` (8 bytes) regardless of how many members are inside `Impl`. Add or remove members → size doesn't change → **ABI stable**.

<br>

#### What IS the compilation firewall?

Without Pimpl:
```
main.cpp → #include "HttpClient.h" → #include <openssl/ssl.h> → 50,000 lines parsed
```

With Pimpl:
```
main.cpp → #include "HttpClient.h" → sees "struct Impl;" → done (no OpenSSL parsed)
```

The header acts as a **firewall** — implementation details don't leak through.

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 6.10 The <code>unique_ptr</code> + Incomplete Type Trap</h2>

There's a critical gotcha. This code looks correct but **won't compile**:

```cpp
// HttpClient.h
class HttpClient {
    struct Impl;
    std::unique_ptr<Impl> pimpl_;
public:
    HttpClient();
    // NO destructor declared — compiler generates one in the HEADER
};
```

The compiler-generated destructor calls `unique_ptr<Impl>::~unique_ptr()`, which calls `delete` on `Impl*`. But `delete` needs the **complete type** (to call `Impl`'s destructor and know its size). At the point where the header generates the destructor, `Impl` is still incomplete → **compile error**.

**Fix**: Declare the destructor in the header, define it in the `.cpp` where `Impl` is complete:

```cpp
// HttpClient.h
class HttpClient {
    struct Impl;
    std::unique_ptr<Impl> pimpl_;
public:
    HttpClient();
    ~HttpClient();   // DECLARED here — not defined (not = default)
};

// HttpClient.cpp
#include "HttpClient.h"

struct HttpClient::Impl { /* ... */ };

HttpClient::HttpClient() : pimpl_(std::make_unique<Impl>()) {}
HttpClient::~HttpClient() = default;   // DEFINED here — Impl is complete
```

The same applies to **move constructor** and **move assignment** — they must be defined in the `.cpp` because `unique_ptr`'s move operations also need the complete type (for the deleter):

```cpp
// HttpClient.h
HttpClient(HttpClient&&) noexcept;
HttpClient& operator=(HttpClient&&) noexcept;

// HttpClient.cpp
HttpClient::HttpClient(HttpClient&&) noexcept = default;
HttpClient& HttpClient::operator=(HttpClient&&) noexcept = default;
```

<br>

#### Memory layout comparison

```
Without Pimpl:
  HttpClient object (on stack or heap):
  ┌─────────────────────────────────────────────────┐
  │ SSL_CTX*          [8 bytes]                     │
  │ CURL*             [8 bytes]                     │
  │ map<str,str>      [48 bytes]                    │
  │ int timeout_ms_   [4 bytes]                     │
  │ bool follow_redir [1 byte + 3 padding]          │
  └─────────────────────────────────────────────────┘
  sizeof(HttpClient) = 72 bytes (varies with members)

With Pimpl:
  HttpClient object (on stack or heap):
  ┌─────────────────────────────────────────────────┐
  │ unique_ptr<Impl>  [8 bytes]  ──────────────►    │
  └─────────────────────────────────────────────────┘
  sizeof(HttpClient) = 8 bytes (ALWAYS — add members to Impl, size unchanged)

                                        On heap:
                                        ┌──────────────────────────┐
                                        │ Impl                     │
                                        │   SSL_CTX*    [8 bytes]  │
                                        │   CURL*       [8 bytes]  │
                                        │   map<...>    [48 bytes] │
                                        │   int         [4 bytes]  │
                                        │   bool        [1 byte]   │
                                        └──────────────────────────┘
```

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 6.11 Pimpl Trade-offs</h2>

| Advantage | Disadvantage |
|-----------|-------------|
| **Fast compilation** — changing Impl only recompiles `.cpp` | **Heap allocation** — Impl is always on the heap (one `new` per object) |
| **ABI stable** — adding members doesn't change sizeof | **Indirection** — every access goes through a pointer (cache miss) |
| **Header clean** — no implementation headers leak | **Boilerplate** — must declare dtor, move ops in header, define in .cpp |
| **Encapsulation** — private members truly hidden | **Debugging** — harder to inspect Impl in debugger |

<br>

#### When to use Pimpl

- **Library/API boundaries** — headers shipped to users (SDK, shared library)
- **Large classes with heavy dependencies** — reduce compile time
- **Stable ABI requirements** — plugin systems, system libraries

<br>

#### When NOT to use Pimpl

- **Performance-critical inner loops** — the extra indirection and heap allocation matter
- **Small/simple classes** — overhead isn't worth it
- **Header-only libraries** — Pimpl requires a `.cpp` file, defeats the purpose
- **Templates** — can't hide template implementation in `.cpp`

<br><br>

---
---

# PART 3: BUILD EXERCISE (60 min)

---

<br>

## Exercise A: COW String

<br>

### Starter skeleton:

```cpp
#include <iostream>
#include <cstring>
#include <utility>

class CowString {
private:
    struct SharedData {
        size_t refcount;
        size_t size;
        char* data;

        SharedData(const char* str, size_t len)
            : refcount(1), size(len), data(new char[len + 1])
        {
            std::memcpy(data, str, len + 1);
        }

        ~SharedData() { delete[] data; }

        void add_ref() { ++refcount; }

        void release() {
            if (--refcount == 0) delete this;
        }
    };

    SharedData* shared_;

    // Detach — make a private copy if shared
    // YOUR IMPLEMENTATION HERE
    void detach() {
        // if refcount > 1:
        //   1. create new SharedData with copy of data
        //   2. decrement old SharedData's refcount
        //   3. point to new SharedData
    }

public:
    // Constructor from C-string
    // YOUR IMPLEMENTATION HERE

    // Destructor
    // YOUR IMPLEMENTATION HERE

    // Copy constructor — CHEAP: just share + increment refcount
    // YOUR IMPLEMENTATION HERE

    // Copy assignment
    // YOUR IMPLEMENTATION HERE

    // Move constructor — steal the pointer
    // YOUR IMPLEMENTATION HERE

    // Move assignment
    // YOUR IMPLEMENTATION HERE

    // operator[] const — read-only, no detach needed
    char operator[](size_t i) const { return shared_->data[i]; }

    // operator[] non-const — MUST detach before returning reference
    // YOUR IMPLEMENTATION HERE

    // c_str() — read-only
    const char* c_str() const { return shared_->data; }

    // size
    size_t size() const { return shared_->size; }

    // refcount (for debugging)
    size_t refcount() const { return shared_->refcount; }

    // append — mutating, needs detach
    // YOUR IMPLEMENTATION HERE
};
```

<br>

### Test driver:

```cpp
int main() {
    std::cout << "=== 1. Construction ===\n";
    CowString a("Hello");
    std::cout << "  a: \"" << a.c_str() << "\" refcount=" << a.refcount() << "\n";

    std::cout << "\n=== 2. Copy — should share (no deep copy) ===\n";
    CowString b = a;
    std::cout << "  a: \"" << a.c_str() << "\" refcount=" << a.refcount() << "\n";
    std::cout << "  b: \"" << b.c_str() << "\" refcount=" << b.refcount() << "\n";
    // refcount should be 2 for both

    std::cout << "\n=== 3. Read via const [] — no detach ===\n";
    const CowString& cr = b;
    char ch = cr[0];
    std::cout << "  cr[0] = '" << ch << "' refcount=" << b.refcount() << "\n";
    // refcount still 2

    std::cout << "\n=== 4. Write via non-const [] — should detach ===\n";
    b[0] = 'J';
    std::cout << "  a: \"" << a.c_str() << "\" refcount=" << a.refcount() << "\n";
    std::cout << "  b: \"" << b.c_str() << "\" refcount=" << b.refcount() << "\n";
    // a should be "Hello" refcount=1, b should be "Jello" refcount=1

    std::cout << "\n=== 5. Multiple copies ===\n";
    CowString c = a;
    CowString d = a;
    CowString e = a;
    std::cout << "  refcount: " << a.refcount() << "\n";   // should be 4

    std::cout << "\n=== 6. One copy modifies — only that one detaches ===\n";
    d[4] = '!';
    std::cout << "  a: \"" << a.c_str() << "\" refcount=" << a.refcount() << "\n";
    std::cout << "  d: \"" << d.c_str() << "\" refcount=" << d.refcount() << "\n";
    // a refcount=3 (a,c,e sharing), d refcount=1

    std::cout << "\n=== 7. append ===\n";
    CowString f("World");
    CowString g = f;
    std::cout << "  before append: f refcount=" << f.refcount() << "\n";
    f.append("!!!");
    std::cout << "  f: \"" << f.c_str() << "\" refcount=" << f.refcount() << "\n";
    std::cout << "  g: \"" << g.c_str() << "\" refcount=" << g.refcount() << "\n";

    std::cout << "\n=== 8. Move — no refcount change ===\n";
    CowString h("Move me");
    CowString i = std::move(h);
    std::cout << "  h: " << (h.c_str() ? h.c_str() : "(null)") << "\n";
    std::cout << "  i: \"" << i.c_str() << "\" refcount=" << i.refcount() << "\n";

    std::cout << "\n=== Destruction ===\n";
    return 0;
}
```

<br>

### Compile and run:

```bash
g++ -std=c++17 -Wall -Wextra -Wpedantic -fsanitize=address -o day06_cow day06_cow.cpp && ./day06_cow
```

<br><br>

---

<br>

## Exercise B: Pimpl — `DatabaseClient`

We'll use `DatabaseClient` instead of `HttpClient` since it doesn't need real curl/openssl — just simulated members.

<br>

### File 1: `DatabaseClient.h` (public header)

```cpp
#pragma once
#include <string>
#include <memory>

class DatabaseClient {
public:
    DatabaseClient(const std::string& host, int port);
    ~DatabaseClient();

    // Move operations — declared here, defined in .cpp
    DatabaseClient(DatabaseClient&&) noexcept;
    DatabaseClient& operator=(DatabaseClient&&) noexcept;

    // No copy
    DatabaseClient(const DatabaseClient&) = delete;
    DatabaseClient& operator=(const DatabaseClient&) = delete;

    // Public interface
    bool connect();
    void disconnect();
    std::string query(const std::string& sql);
    bool is_connected() const;

private:
    struct Impl;                       // forward declaration only
    std::unique_ptr<Impl> pimpl_;      // opaque pointer
};
```

Notice: **no implementation headers** — no map, no vector, no socket includes. Users of `DatabaseClient.h` don't see any of that.

<br>

### File 2: `DatabaseClient.cpp` (private implementation)

```cpp
#include "DatabaseClient.h"
#include <iostream>
#include <map>
#include <vector>
#include <sstream>

// The actual implementation — hidden from all users of the header
struct DatabaseClient::Impl {
    std::string host;
    int port;
    bool connected = false;
    int query_count = 0;
    std::map<std::string, std::string> cache;
    std::vector<std::string> query_log;

    Impl(const std::string& h, int p) : host(h), port(p) {}
};

DatabaseClient::DatabaseClient(const std::string& host, int port)
    : pimpl_(std::make_unique<Impl>(host, port))
{
    std::cout << "  [DatabaseClient] created for " << host << ":" << port << "\n";
}

DatabaseClient::~DatabaseClient() {
    if (pimpl_ && pimpl_->connected) {
        std::cout << "  [DatabaseClient] auto-disconnecting\n";
    }
}

DatabaseClient::DatabaseClient(DatabaseClient&&) noexcept = default;
DatabaseClient& DatabaseClient::operator=(DatabaseClient&&) noexcept = default;

bool DatabaseClient::connect() {
    pimpl_->connected = true;
    std::cout << "  [DatabaseClient] connected to "
              << pimpl_->host << ":" << pimpl_->port << "\n";
    return true;
}

void DatabaseClient::disconnect() {
    pimpl_->connected = false;
    std::cout << "  [DatabaseClient] disconnected\n";
}

std::string DatabaseClient::query(const std::string& sql) {
    if (!pimpl_->connected) return "ERROR: not connected";

    pimpl_->query_count++;
    pimpl_->query_log.push_back(sql);

    // Check cache
    auto it = pimpl_->cache.find(sql);
    if (it != pimpl_->cache.end()) {
        std::cout << "  [query] cache hit: " << sql << "\n";
        return it->second;
    }

    // Simulate query
    std::ostringstream result;
    result << "Result for: " << sql << " (query #" << pimpl_->query_count << ")";
    pimpl_->cache[sql] = result.str();
    std::cout << "  [query] executed: " << sql << "\n";
    return result.str();
}

bool DatabaseClient::is_connected() const {
    return pimpl_->connected;
}
```

<br>

### File 3: `main.cpp` (test)

```cpp
#include "DatabaseClient.h"
#include <iostream>

int main() {
    std::cout << "=== 1. Create and connect ===\n";
    DatabaseClient db("localhost", 5432);
    db.connect();

    std::cout << "\n=== 2. Query ===\n";
    std::cout << "  " << db.query("SELECT * FROM users") << "\n";
    std::cout << "  " << db.query("SELECT * FROM users") << "\n";  // cache hit

    std::cout << "\n=== 3. Move ===\n";
    DatabaseClient db2 = std::move(db);
    std::cout << "  db2 connected: " << db2.is_connected() << "\n";
    std::cout << "  " << db2.query("SELECT count(*) FROM orders") << "\n";

    std::cout << "\n=== 4. sizeof check ===\n";
    std::cout << "  sizeof(DatabaseClient) = " << sizeof(DatabaseClient) << "\n";
    // Should be 8 (just the unique_ptr)

    std::cout << "\n=== Destruction ===\n";
    return 0;
}
```

<br>

### Compile (multi-file):

```bash
g++ -std=c++17 -Wall -Wextra -c DatabaseClient.cpp -o DatabaseClient.o
g++ -std=c++17 -Wall -Wextra -c main.cpp -o main.o
g++ DatabaseClient.o main.o -o day06_pimpl && ./day06_pimpl
```

Key observation: if you change `Impl`'s members (add a field, change types), only `DatabaseClient.cpp` recompiles. `main.cpp` doesn't need to be touched.

<br><br>

---
---

# PART 4: DEEP DIVE Q&A

---

<br>

<h2 style="color: #2980B9;">📘 Q1: Why was COW removed from <code>std::string</code> in C++11?</h2>

Three reasons:

**1. Thread safety cost**: With COW, even a "read-only" `operator[]` might trigger a detach (decrement refcount). In multithreaded code, refcount operations must be atomic. Adding atomic overhead to every `string` operation was unacceptable.

**2. Iterator/reference invalidation**: With COW, a non-const `operator[]` detaches, which reallocates the buffer. Any existing iterators or pointers into the string are invalidated:

```cpp
std::string s = "Hello";
char& ref = s[0];    // ref points into s's buffer
std::string t = s;   // COW: share buffer
s[0] = 'J';          // detach! s allocates new buffer
// ref now points to t's buffer, not s's — SURPRISE
```

**3. Small String Optimization (SSO) is better**: Modern `std::string` stores short strings (typically ≤ 22 chars) directly inside the object — no heap allocation at all. For short strings, SSO beats COW because there's no refcount overhead and no heap allocation even for copies.

```
SSO layout (typical):
  std::string object (on stack):
  ┌─────────────────────────────────────────┐
  │ union {                                 │
  │   char short_buf[23];  ← short strings  │
  │   struct { char* ptr; size_t len,cap; } │
  │ }                                       │
  │ size_t size_or_flag;                    │
  └─────────────────────────────────────────┘
```

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 Q2: Can I use <code>shared_ptr</code> instead of manual refcounting for COW?</h2>

Yes! In fact it's much simpler:

```cpp
class CowString {
    std::shared_ptr<std::string> data_;

    void detach() {
        if (data_.use_count() > 1) {
            data_ = std::make_shared<std::string>(*data_);  // deep copy
        }
    }
public:
    CowString(const char* s) : data_(std::make_shared<std::string>(s)) {}

    char& operator[](size_t i) { detach(); return (*data_)[i]; }
    char operator[](size_t i) const { return (*data_)[i]; }
};
```

This gives you thread-safe refcounting for free (atomic in `shared_ptr`). The manual version in the exercise is to understand the mechanics — in production code, use `shared_ptr` unless you need more control.

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 Q3: Why does Pimpl need <code>unique_ptr</code> and not a raw pointer?</h2>

You could use a raw pointer:

```cpp
class HttpClient {
    struct Impl;
    Impl* pimpl_;   // raw pointer
public:
    HttpClient() : pimpl_(new Impl()) {}
    ~HttpClient() { delete pimpl_; }
    // must also handle copy/move manually...
};
```

But this is exactly what we learned on Day 2 — raw pointers have no ownership semantics. With `unique_ptr`:

- Destructor is auto-generated (in the `.cpp`)
- Move operations are auto-generated (in the `.cpp`)
- No chance of forgetting `delete`
- Exception-safe (if constructor throws after `pimpl_` is assigned, `unique_ptr` cleans up)

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 Q4: What is the "fast Pimpl" optimization?</h2>

The main cost of Pimpl is the heap allocation for `Impl`. The "fast Pimpl" avoids this by storing `Impl` in a pre-sized stack buffer:

```cpp
class HttpClient {
    static constexpr size_t ImplSize = 128;   // big enough for Impl
    static constexpr size_t ImplAlign = 8;

    alignas(ImplAlign) unsigned char storage_[ImplSize];

    Impl* pimpl() { return reinterpret_cast<Impl*>(storage_); }

public:
    HttpClient() { new (storage_) Impl(); }          // placement new
    ~HttpClient() { pimpl()->~Impl(); }               // explicit destructor call
};
```

This avoids the heap allocation but requires you to know `sizeof(Impl)` at compile time and hardcode it. If `Impl`'s size changes, you need to update `ImplSize`. Some implementations add a `static_assert` in the `.cpp` to catch size mismatches.

This is an advanced optimization — use regular Pimpl with `unique_ptr` unless profiling shows the allocation matters.

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 Q5: Pimpl with <code>shared_ptr</code> — what changes?</h2>

If you use `shared_ptr` instead of `unique_ptr` for Pimpl:

```cpp
class HttpClient {
    struct Impl;
    std::shared_ptr<Impl> pimpl_;   // shared, not unique
};
```

Differences:

1. **Copying is allowed** — `HttpClient` copies share the same `Impl` (like COW without the write detection)
2. **No need to define dtor/move in .cpp** — `shared_ptr` uses type erasure for the deleter, so it doesn't need the complete type at the point of destruction in the header
3. **Overhead** — `shared_ptr` has a control block + atomic refcount

This is what **Qt** does with its `QExplicitlySharedDataPointer`. It's essentially COW + Pimpl combined.

<br><br>

---
---

# PART 5: REFLECT & NOTES (10 min)

---

<br>

1. **What is COW in one sentence?**
   - Share data on copy, only duplicate when someone writes.

2. **Why was COW removed from `std::string`?**
   - Thread safety costs, iterator invalidation on detach, SSO is better for short strings.

3. **What is Pimpl in one sentence?**
   - Hide all private members behind an opaque pointer to reduce coupling, compile time, and ABI breakage.

4. **Why must the destructor be defined in the `.cpp` with Pimpl?**
   - `unique_ptr`'s destructor needs the complete type to call `delete`. In the header, `Impl` is incomplete.

5. **When is COW still useful?**
   - Large immutable data (configs, documents), OS memory pages (fork), undo systems, functional data structures.

<br><br>

---
---

# INTERVIEW QUESTIONS TO PRACTICE

---

<br>

1. "What is Copy-on-Write? When would you use it?"
2. "Why doesn't `std::string` use COW anymore?"
3. "Implement a COW string with a shared reference-counted buffer."
4. "What is the Pimpl idiom? What problems does it solve?"
5. "Why do you need to declare the destructor in the header when using Pimpl with `unique_ptr`?"
6. "What is ABI stability and how does Pimpl help?"
7. "What are the trade-offs of Pimpl?"
8. "How does Qt use implicit sharing (COW + Pimpl)?"

<br><br>

---
---

# REFERENCE SOLUTION

---

<br>

<details>
<summary>Click to expand the complete CowString implementation</summary>

<br>

```cpp
#include <iostream>
#include <cstring>
#include <utility>
#include <cassert>

class CowString {
    struct SharedData {
        size_t refcount;
        size_t size;
        char* data;

        SharedData(const char* str, size_t len)
            : refcount(1), size(len), data(new char[len + 1])
        {
            std::memcpy(data, str, len + 1);
            std::cout << "  [SharedData] allocated \"" << data << "\"\n";
        }

        ~SharedData() {
            std::cout << "  [SharedData] freed \"" << data << "\"\n";
            delete[] data;
        }

        void add_ref() { ++refcount; }

        void release() {
            if (--refcount == 0) delete this;
        }
    };

    SharedData* shared_;

    void detach() {
        if (shared_->refcount == 1) return;
        SharedData* new_data = new SharedData(shared_->data, shared_->size);
        shared_->release();
        shared_ = new_data;
        std::cout << "  [detach] made private copy\n";
    }

public:
    CowString(const char* str = "")
        : shared_(new SharedData(str, std::strlen(str))) {}

    ~CowString() {
        if (shared_) shared_->release();
    }

    CowString(const CowString& other) : shared_(other.shared_) {
        shared_->add_ref();
        std::cout << "  [copy] sharing, refcount=" << shared_->refcount << "\n";
    }

    CowString& operator=(const CowString& other) {
        if (this == &other) return *this;
        shared_->release();
        shared_ = other.shared_;
        shared_->add_ref();
        return *this;
    }

    CowString(CowString&& other) noexcept : shared_(other.shared_) {
        other.shared_ = nullptr;
    }

    CowString& operator=(CowString&& other) noexcept {
        if (this == &other) return *this;
        if (shared_) shared_->release();
        shared_ = other.shared_;
        other.shared_ = nullptr;
        return *this;
    }

    char operator[](size_t i) const { return shared_->data[i]; }

    char& operator[](size_t i) {
        detach();
        return shared_->data[i];
    }

    const char* c_str() const { return shared_ ? shared_->data : ""; }
    size_t size() const { return shared_ ? shared_->size : 0; }
    size_t refcount() const { return shared_ ? shared_->refcount : 0; }

    void append(const char* str) {
        detach();
        size_t add_len = std::strlen(str);
        char* new_buf = new char[shared_->size + add_len + 1];
        std::memcpy(new_buf, shared_->data, shared_->size);
        std::memcpy(new_buf + shared_->size, str, add_len + 1);
        delete[] shared_->data;
        shared_->data = new_buf;
        shared_->size += add_len;
    }
};

int main() {
    std::cout << "=== 1. Construction ===\n";
    CowString a("Hello");
    std::cout << "  a: \"" << a.c_str() << "\" ref=" << a.refcount() << "\n";

    std::cout << "\n=== 2. Copy (cheap — shared) ===\n";
    CowString b = a;
    std::cout << "  a ref=" << a.refcount() << "  b ref=" << b.refcount() << "\n";

    std::cout << "\n=== 3. Const read (no detach) ===\n";
    const CowString& cr = b;
    std::cout << "  cr[0]='" << cr[0] << "' ref=" << b.refcount() << "\n";

    std::cout << "\n=== 4. Write (detach!) ===\n";
    b[0] = 'J';
    std::cout << "  a: \"" << a.c_str() << "\" ref=" << a.refcount() << "\n";
    std::cout << "  b: \"" << b.c_str() << "\" ref=" << b.refcount() << "\n";

    std::cout << "\n=== 5. Multiple copies ===\n";
    CowString c = a, d = a, e = a;
    std::cout << "  ref=" << a.refcount() << "\n";

    d[4] = '!';
    std::cout << "  a ref=" << a.refcount() << "  d ref=" << d.refcount() << "\n";

    std::cout << "\n=== 6. Append ===\n";
    CowString f("World");
    CowString g = f;
    f.append("!!!");
    std::cout << "  f: \"" << f.c_str() << "\" ref=" << f.refcount() << "\n";
    std::cout << "  g: \"" << g.c_str() << "\" ref=" << g.refcount() << "\n";

    std::cout << "\n=== Done ===\n";
    return 0;
}
```

</details>

<br>

---

**Next**: [Day 7 — Weekly Review →](day07-weekly-review.md)  *(review day — no new material)*
