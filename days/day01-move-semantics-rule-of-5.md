# Day 1: Move Semantics & Rule of 5

[← Back to Study Plan](../lld-study-plan.md)

> **Time**: ~2 hours
> **Goal**: Deeply understand value categories, move semantics, and the Rule of 5. Be able to write a class with correct copy/move behavior from scratch.

---
---

# PART 1: THEORY (30 min)

---

<br>

<h2 style="color: #2980B9;">📘 1.1 Value Categories — lvalue vs rvalue</h2>

Every expression in C++ is either an **lvalue** (has an identity, can take its address) or an **rvalue** (temporary, about to be destroyed).

```cpp
int x = 42;        // x is an lvalue
int y = x + 1;     // (x + 1) is an rvalue — a temporary

std::string s = "hello";           // s is an lvalue
std::string t = s + " world";     // (s + " world") is an rvalue
```

**Why this matters**: If a value is about to die (rvalue), we can *steal* its resources instead of copying them. This is the core insight behind move semantics.

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 1.2 Lvalue References vs Rvalue References</h2>

```cpp
int& ref = x;       // lvalue reference — binds to lvalues
int&& rref = 42;    // rvalue reference — binds to rvalues

// A function can distinguish between the two:
void process(std::string& s);      // called with lvalues
void process(std::string&& s);     // called with rvalues (temporaries)
```

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 1.3 What <code>std::move</code> Actually Does</h2>

`std::move` does **NOT** move anything. It is a **cast** — it converts an lvalue into an rvalue reference, signaling "I'm done with this, you may steal from it."

```cpp
// Simplified implementation of std::move
template <typename T>
typename std::remove_reference<T>::type&& move(T&& arg) {
    return static_cast<typename std::remove_reference<T>::type&&>(arg);
}
```

```cpp
std::string a = "hello";
std::string b = std::move(a);  // a is cast to rvalue → string's move constructor steals a's buffer
// a is now in a valid-but-unspecified state (likely empty)
```

**Critical rule**: After `std::move`, the source object must be in a **valid but unspecified** state. You can destroy it or assign to it, but don't read from it.

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 1.4 The Special Member Functions</h2>

C++ can auto-generate up to 6 special members:

| Function | Syntax | Purpose |
|----------|--------|---------|
| Default constructor | `T()` | Construct with no arguments |
| Destructor | `~T()` | Clean up resources |
| Copy constructor | `T(const T&)` | Create a copy |
| Copy assignment | `T& operator=(const T&)` | Assign from a copy |
| Move constructor | `T(T&&)` | Steal resources from a temporary |
| Move assignment | `T& operator=(T&&)` | Steal-assign from a temporary |

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 1.5 The Rule of 5</h2>

> If you define **any** of: destructor, copy constructor, copy assignment, move constructor, or move assignment — you should **define all 5**.

**Why?** If your class manages a resource (raw pointer, file handle, socket), the compiler-generated versions will do the wrong thing (shallow copy, double free, etc).

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 1.6 When Does the Compiler Generate Them?</h2>

This is subtle and frequently asked in interviews:

```
                        User declares:
                  ┌──────────────────────────────────┐
                  │ dtor  cc    ca    mc    ma       │
Auto-generates:   │                                  │
  Destructor      │  ✗    ✓     ✓     ✓     ✓       │
  Copy ctor       │  ✓*   ✗     ✓*    ✗     ✗       │
  Copy assign     │  ✓*   ✓*    ✗     ✗     ✗       │
  Move ctor       │  ✗    ✗     ✗     ✗     ✗       │
  Move assign     │  ✗    ✗     ✗     ✗     ✗       │
                  └──────────────────────────────────┘

✓  = auto-generated
✓* = auto-generated but DEPRECATED (may be removed in future standards)
✗  = NOT auto-generated (deleted or not declared)
```

**The core rules in plain English:**

1. **Move operations are fragile** — the compiler will only auto-generate them if you haven't declared *any* of: destructor, copy constructor, or copy assignment. The moment you write any one of those three, the compiler assumes you have custom resource logic and refuses to guess how to move.

2. **Copy operations are stubborn (but deprecated)** — for backward compatibility with pre-C++11 code, the compiler still auto-generates copy ctor/copy assignment even if you declare a destructor. But this behavior is **deprecated** and may be removed in a future standard. Never rely on it.

3. **Declaring either move operation kills both copy operations** — if you write a move constructor, the compiler deletes the copy constructor (and vice versa). The reasoning: if you customized move, you probably need custom copy too.

<br>

#### Example 1: The "innocent destructor" trap

```cpp
class FileHandle {
    int fd_;
public:
    FileHandle(int fd) : fd_(fd) {}
    ~FileHandle() { ::close(fd_); }  // just added a destructor...
};

FileHandle a(open("file.txt", O_RDONLY));
FileHandle b = std::move(a);  // SURPRISE: this calls the COPY constructor!
// Now both a and b have the same fd_ → double close → undefined behavior
```

**What happened?** You declared a destructor, so the compiler **suppressed** move constructor generation. There's no move constructor, so `std::move(a)` falls back to the copy constructor (an rvalue reference *can* bind to `const T&`). Both objects now hold the same file descriptor.

**Fix:** Explicitly declare all 5:

```cpp
class FileHandle {
    int fd_;
public:
    FileHandle(int fd) : fd_(fd) {}
    ~FileHandle() { if (fd_ >= 0) ::close(fd_); }

    FileHandle(const FileHandle&) = delete;             // no copying
    FileHandle& operator=(const FileHandle&) = delete;

    FileHandle(FileHandle&& other) noexcept : fd_(other.fd_) {
        other.fd_ = -1;  // source no longer owns it
    }
    FileHandle& operator=(FileHandle&& other) noexcept {
        if (this != &other) {
            if (fd_ >= 0) ::close(fd_);
            fd_ = other.fd_;
            other.fd_ = -1;
        }
        return *this;
    }
};
```

<br>

#### Example 2: Declaring copy constructor kills moves

```cpp
class Buffer {
    std::vector<char> data_;
public:
    Buffer(const Buffer& other) : data_(other.data_) {
        std::cout << "deep copy\n";
    }
    // No move ctor declared → compiler does NOT auto-generate one
};

Buffer makeBuffer() {
    Buffer b;
    return b;  // without RVO, this would COPY, not move
}
```

Because you declared a copy constructor, the compiler **will not** generate a move constructor. All "moves" silently fall back to copies.

<br>

#### Example 3: The fully correct class

```cpp
class Widget {
    int* data_;
public:
    Widget() : data_(new int(0)) {}

    // You wrote a destructor → you MUST write the other 4
    ~Widget() { delete data_; }

    Widget(const Widget& other) : data_(new int(*other.data_)) {}   // deep copy
    Widget& operator=(const Widget& other) {                         // copy assign
        if (this != &other) { *data_ = *other.data_; }
        return *this;
    }
    Widget(Widget&& other) noexcept : data_(other.data_) {          // steal
        other.data_ = nullptr;
    }
    Widget& operator=(Widget&& other) noexcept {                    // steal-assign
        if (this != &other) { delete data_; data_ = other.data_; other.data_ = nullptr; }
        return *this;
    }
};
```

<br>

#### Quick verification trick

Not sure if the compiler generated a move constructor? Try this:

```cpp
#include <type_traits>

static_assert(std::is_move_constructible_v<MyClass>,    "move ctor exists");
static_assert(std::is_move_assignable_v<MyClass>,       "move assign exists");
static_assert(std::is_nothrow_move_constructible_v<MyClass>, "move ctor is noexcept");
```

If these fail at compile time, you know the compiler didn't generate (or you didn't declare) the move operations.

**Key takeaway**: Declaring a destructor suppresses move operations. Declaring any copy/move operation suppresses auto-generation of the other move operations. This is why you follow the Rule of 5.

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 1.7 <code>= default</code> and <code>= delete</code></h2>

```cpp
class Widget {
public:
    Widget() = default;                          // let compiler generate
    Widget(const Widget&) = delete;              // explicitly disallow copying
    Widget& operator=(const Widget&) = delete;
    Widget(Widget&&) = default;                  // explicitly request move
    Widget& operator=(Widget&&) = default;
};
```

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 1.8 Copy-and-Swap Idiom</h2>

A robust pattern to implement assignment operators exception-safely:

```cpp
class String {
    char* data_;
    size_t size_;

public:
    // Swap helper (noexcept!)
    friend void swap(String& a, String& b) noexcept {
        using std::swap;
        swap(a.data_, b.data_);
        swap(a.size_, b.size_);
    }

    // Single assignment operator handles BOTH copy and move assignment
    String& operator=(String other) {  // note: passed by VALUE
        swap(*this, other);            // swap with the copy/move
        return *this;                  // old data destroyed when 'other' goes out of scope
    }
};
```

<br>

#### But wait — the parameter is always destroyed, so isn't it always a move?

No. The key insight is: **the copy-or-move happens BEFORE the function body even starts — in the parameter construction**. The swap and destruction inside the function are always the same. What differs is *how `other` gets created* in the first place.

Let's trace through both cases step by step:

<br>

#### Case 1: Called with an lvalue (copy path)

```cpp
String a("Hello");     // a owns: data_ → "Hello"
String b("World");     // b owns: data_ → "World"

b = a;                 // a is an lvalue
```

Here's what happens at `b = a`:

```
Step 1: PARAMETER CONSTRUCTION (this is where copy happens)
   a is an lvalue → compiler calls String(const String&) copy constructor
   to create 'other'.
   other.data_ → new allocation → "Hello" (deep copy of a)

   State now:
     a.data_     → "Hello"  (untouched)
     b.data_     → "World"  (about to be replaced)
     other.data_ → "Hello"  (fresh copy)

Step 2: swap(*this, other)
   Swaps b's pointers with other's pointers.

   State now:
     a.data_     → "Hello"  (untouched)
     b.data_     → "Hello"  (was other's — the fresh copy)
     other.data_ → "World"  (was b's — the old data)

Step 3: 'other' goes out of scope → destructor runs
   Frees "World" (b's OLD data). Exactly what we wanted.

Result: a still has "Hello", b now has its own copy of "Hello". ✓
```

<br>

#### Case 2: Called with an rvalue (move path)

```cpp
String a("Hello");     // a owns: data_ → "Hello"
String b("World");     // b owns: data_ → "World"

b = std::move(a);      // std::move(a) is an rvalue
```

Here's what happens at `b = std::move(a)`:

```
Step 1: PARAMETER CONSTRUCTION (this is where move happens)
   std::move(a) is an rvalue → compiler calls String(String&&) move constructor
   to create 'other'.
   other STEALS a's pointer. No allocation, no copy.

   State now:
     a.data_     → nullptr  (gutted — moved from)
     b.data_     → "World"  (about to be replaced)
     other.data_ → "Hello"  (stolen from a — zero cost)

Step 2: swap(*this, other)
   Swaps b's pointers with other's pointers.

   State now:
     a.data_     → nullptr  (still empty)
     b.data_     → "Hello"  (was other's — stolen from a)
     other.data_ → "World"  (was b's — the old data)

Step 3: 'other' goes out of scope → destructor runs
   Frees "World" (b's OLD data).

Result: a is empty, b has "Hello" with zero allocations. ✓
```

<br>

#### Side-by-side comparison

```
                    COPY PATH (b = a)         MOVE PATH (b = std::move(a))
                    ─────────────────         ──────────────────────────────
Parameter 'other'   copy-constructed          move-constructed
  created by:       (new allocation +         (just pointer steal,
                     memcpy = EXPENSIVE)       no alloc = CHEAP)

swap(*this,other)   same in both cases        same in both cases

other destroyed     same in both cases        same in both cases
```

**The function body (swap + destroy) is identical both times.** The only difference is Step 1 — how `other` gets constructed. That's where the compiler picks copy ctor vs move ctor based on whether you passed an lvalue or rvalue.

<br>

#### Why is this better than writing two separate operators?

1. **Exception safety for free** — if the copy constructor throws (e.g., `new` fails in Step 1), `*this` hasn't been touched yet. No partial state corruption.
2. **Self-assignment safe** — `a = a` works: `other` gets a copy of `a`, swap exchanges them, destructor frees the old copy. No special `if (this != &other)` check needed.
3. **Less code** — one function instead of two, fewer places for bugs.

<br>

#### Trade-off

The copy path always creates a full copy (even for self-assignment `a = a`, where a hand-written operator could just `return *this`). In practice this rarely matters — self-assignment is uncommon, and the simplicity/safety wins.

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 1.9 Why Is <code>swap</code> Declared <code>friend</code>? (ADL & Hidden Friends)</h2>

You might wonder — why not just make `swap` a normal member function? The answer involves **ADL** and how generic C++ code discovers your custom swap.

<br>

#### What is ADL (Argument-Dependent Lookup)?

When you call a function **without** a namespace qualifier, the compiler doesn't just search the current scope — it **also searches the namespace where the argument types are defined**.

```cpp
namespace graphics {
    class Point { int x, y; };
    void draw(Point p) { /* ... */ }
}

int main() {
    graphics::Point p;
    draw(p);    // No "graphics::" prefix needed!
                // Compiler sees p is graphics::Point
                // → searches namespace graphics
                // → finds draw(Point) ✓
}
```

Without ADL you'd have to write `graphics::draw(p)` every time. ADL makes unqualified calls work when the function "belongs" to the type's namespace.

<br>

#### Why does this matter for `swap`?

The STL and generic code use this pattern to find the best swap:

```cpp
template <typename T>
void doSomething(T& a, T& b) {
    using std::swap;   // make std::swap available as a fallback
    swap(a, b);        // UNQUALIFIED call → ADL kicks in
                       // if T has its own swap in its namespace → uses that (optimized)
                       // otherwise → falls back to std::swap (3 moves)
}
```

If `T` is your `String` and you have a `friend void swap(String&, String&)`, ADL finds your fast pointer-swap. Without it, the code falls back to `std::swap` which does a move-construct + two move-assigns (works, but less optimal for complex types).

<br>

#### Why a member function doesn't work

ADL only finds **free functions**, not member functions:

```cpp
class String {
public:
    void swap(String& other) noexcept { /* ... */ }   // member function
};

String a("Hello"), b("World");
swap(a, b);     // ERROR: no free function "swap" found
a.swap(b);      // works, but generic code won't call it this way
```

Generic code always calls `swap(a, b)`, never `a.swap(b)`. So a member function is invisible to it.

<br>

#### The hidden friend idiom

When you define a `friend` function **inside** the class body, it becomes a free function in the enclosing namespace, but with a twist — it's **only discoverable through ADL**:

```cpp
class String {
    char* data_;
    size_t size_;

    friend void swap(String& a, String& b) noexcept {
        using std::swap;
        swap(a.data_, b.data_);
        swap(a.size_, b.size_);
    }
};
```

This function is:
- **Not** a member of `String` — `String::swap(a, b)` will not compile
- **Not** visible through normal lookup — just writing `swap(...)` without a `String` argument won't find it
- **Only** found through ADL — when you pass a `String` argument to unqualified `swap()`

This is called a "hidden friend" because:
- `friend` → it can access `data_` and `size_` (private members)
- "hidden" → it only exists for ADL, no namespace pollution

<br>

#### Three ways to write swap — compared

```cpp
// Option 1: Member function
class String {
public:
    void swap(String& other);
};
// ✗ ADL can't find it — generic code won't use it
// ✓ Can access private members

// Option 2: Free function with separate friend declaration
class String {
    friend void swap(String&, String&);
};
void swap(String& a, String& b) { /* ... */ }
// ✓ ADL finds it
// ✓ Also visible to normal lookup (anyone can see it)
// ✓ Can access private members

// Option 3: Hidden friend (defined inside class body)
class String {
    friend void swap(String& a, String& b) { /* ... */ }
};
// ✓ ADL finds it
// ✗ NOT visible to normal lookup (only through ADL — cleanest)
// ✓ Can access private members
// ★ Preferred modern C++ style
```

**Option 3 is what we use in the copy-and-swap idiom.** It's the cleanest — the function lives right next to the data it touches, it's found by generic code via ADL, and it doesn't pollute the namespace.

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 1.10 Custom Swap vs Generic <code>std::swap</code> — What's Actually Faster?</h2>

You might think: "our custom swap calls `std::swap` on individual members anyway, so doesn't it also do 3 moves? What's the point?"

The difference is **what** is being moved.

<br>

#### Our custom swap — swaps raw members directly

```cpp
friend void swap(String& a, String& b) noexcept {
    using std::swap;
    swap(a.data_, b.data_);   // swap two char* pointers (8 bytes)
    swap(a.size_, b.size_);   // swap two size_t values (8 bytes)
}
```

Each `std::swap` here operates on a **primitive type** (pointer, integer). Three moves of an 8-byte value = just copying 24 bytes per member. No constructors, no destructors, no logic. Just shuffling bytes.

Total cost: **6 moves of primitives. Zero calls to String's special members. Zero allocations.**

<br>

#### Generic `std::swap` on the whole String — goes through your constructors

If ADL doesn't find your custom swap, the generic `std::swap<String>` is used:

```cpp
// What std::swap does internally:
template <typename T>
void swap(T& a, T& b) {
    T temp = std::move(a);    // calls String(String&&) move constructor
    a = std::move(b);         // calls String::operator=(String&&) move assignment
    b = std::move(temp);      // calls String::operator=(String&&) move assignment
    // ~temp()                // calls String destructor
}
```

This invokes your **String move constructor**, **move assignment** (twice), and **destructor** — full function calls with null-checking, `delete`-ing old data, nulling out the source, etc.

<br>

#### The real danger — infinite recursion with copy-and-swap

If your `operator=` uses copy-and-swap internally, and you **don't** have a custom swap, you get infinite recursion:

```
std::swap<String>(a, b)
  → calls: a = std::move(b)              // move assignment
    → operator=(String other)             // other is move-constructed
      → calls: swap(*this, other)         // no custom swap found...
        → calls: std::swap<String>(...)   // falls back to generic
          → calls: *this = std::move(other)   // move assignment AGAIN
            → operator=(String other)
              → swap(*this, other)
                → std::swap<String>(...)
                  → ... INFINITE RECURSION → stack overflow
```

**The custom swap is NOT optional when using copy-and-swap. Without it, your program crashes.**

<br>

#### Side-by-side comparison

```
Custom swap                          Generic std::swap<String>
──────────────                       ─────────────────────────
swap(a.data_, b.data_)               String temp = std::move(a)
swap(a.size_, b.size_)                 → move ctor: steal ptr, null source
                                     a = std::move(b)
                                       → move assign: delete old, steal, null
                                     b = std::move(temp)
                                       → move assign: delete old, steal, null
                                     ~temp()
                                       → destructor (no-op on nullptr)

Calls to String special members: 0   Calls to String special members: 4
Allocations: 0                        Allocations: 0
Can infinite-recurse: No              YES (if operator= uses swap inside)
```

<br>

#### Summary: three reasons the custom swap exists

1. **Avoids infinite recursion** when combined with copy-and-swap (critical, not optional)
2. **Bypasses all String machinery** — no constructors, no destructors, no null checks, no `delete`. Just raw pointer/integer swaps
3. **Trivially `noexcept`** — swapping primitives can never throw. The generic version is only `noexcept` if your move constructor and move assignment are both `noexcept`

For a simple class like `String` with two members, the performance difference is small. But for a class with many members, expensive destructors, or one that uses copy-and-swap — the custom swap is either significantly faster or **the only thing that prevents a crash**.

<br><br>

---
---

# PART 2: BUILD EXERCISE (60 min)

---

<br>

## Exercise: Implement a `String` Class with Full Rule of 5

Build a heap-allocated string class that demonstrates every special member function. Add `std::cout` prints in each so you can observe exactly when they're called.

<br>

### Starter skeleton:

```cpp
#include <iostream>
#include <cstring>
#include <utility>  // std::swap, std::move

class String {
private:
    char* data_;
    size_t size_;

public:
    // --- Constructor ---
    String(const char* str = "") {
        size_ = std::strlen(str);
        data_ = new char[size_ + 1];
        std::memcpy(data_, str, size_ + 1);
        std::cout << "Constructor: \"" << data_ << "\"\n";
    }

    // --- Destructor ---
    ~String() {
        std::cout << "Destructor: \"" << (data_ ? data_ : "null") << "\"\n";
        delete[] data_;
    }

    // --- Copy Constructor ---
    // YOUR IMPLEMENTATION HERE
    // Should deep-copy data_ from other

    // --- Copy Assignment (using copy-and-swap) ---
    // YOUR IMPLEMENTATION HERE

    // --- Move Constructor ---
    // YOUR IMPLEMENTATION HERE
    // Should steal data_ from other and null out other's pointer

    // --- Move Assignment ---
    // YOUR IMPLEMENTATION HERE

    // --- Swap ---
    friend void swap(String& a, String& b) noexcept {
        using std::swap;
        swap(a.data_, b.data_);
        swap(a.size_, b.size_);
    }

    // --- Accessors ---
    const char* c_str() const { return data_; }
    size_t size() const { return size_; }

    // --- Concatenation (returns rvalue — triggers move) ---
    friend String operator+(const String& lhs, const String& rhs) {
        char* buf = new char[lhs.size_ + rhs.size_ + 1];
        std::memcpy(buf, lhs.data_, lhs.size_);
        std::memcpy(buf + lhs.size_, rhs.data_, rhs.size_ + 1);
        String result;
        delete[] result.data_;
        result.data_ = buf;
        result.size_ = lhs.size_ + rhs.size_;
        return result;
    }
};
```

<br>

### Test driver to verify behavior:

```cpp
int main() {
    std::cout << "=== 1. Construction ===\n";
    String a("Hello");

    std::cout << "\n=== 2. Copy construction ===\n";
    String b = a;  // copy ctor

    std::cout << "\n=== 3. Copy assignment ===\n";
    String c("World");
    c = a;  // copy assignment

    std::cout << "\n=== 4. Move construction ===\n";
    String d = std::move(a);  // move ctor — a is now empty

    std::cout << "\n=== 5. Move assignment ===\n";
    String e("Temp");
    e = std::move(b);  // move assignment — b is now empty

    std::cout << "\n=== 6. Move from return value ===\n";
    String f = String("Temporary");  // may be elided (RVO), use -fno-elide-constructors to force

    std::cout << "\n=== 7. Concatenation (triggers move) ===\n";
    String g = d + c;  // operator+ returns rvalue → move ctor

    std::cout << "\n=== 8. Self-assignment safety ===\n";
    g = g;  // must not crash!

    std::cout << "\n=== Destruction (reverse order) ===\n";
    return 0;
}
```

<br>

### Compile and run:

```bash
g++ -std=c++17 -Wall -Wextra -Wpedantic -fsanitize=address -o day01 day01_string.cpp && ./day01
```

### To force seeing moves (disable RVO):

```bash
g++ -std=c++17 -fno-elide-constructors -fsanitize=address -o day01_no_rvo day01_string.cpp && ./day01_no_rvo
```

<br>

### Expected Output Pattern

You should see each special member being called at the right time. For example:

```
=== 1. Construction ===
Constructor: "Hello"

=== 2. Copy construction ===
Copy Constructor: "Hello"

=== 3. Copy assignment ===
Constructor: "World"
Copy Assignment: "Hello"

=== 4. Move construction ===
Move Constructor: "Hello" (stolen from source)

=== 5. Move assignment ===
Constructor: "Temp"
Move Assignment: "Hello" (stolen from source)
...
```

<br><br>

---
---

# PART 3: DEEP DIVE CHALLENGES (20 min)

---

<br>

Once your basic `String` class works, tackle these:

<br>

## Challenge 1: Make it `noexcept`-correct

Move operations should be `noexcept`. STL containers (like `std::vector`) will **fall back to copy** if your move constructor is not `noexcept`.

```cpp
String(String&& other) noexcept;
String& operator=(String&& other) noexcept;
```

**Verify**: Put your `String` in a `std::vector`, call `push_back` several times, and confirm move constructor is used during reallocation (not copy).

<br>

## Challenge 2: Unified assignment with copy-and-swap

Replace separate copy assignment and move assignment with a single:

```cpp
String& operator=(String other) noexcept {  // by value!
    swap(*this, other);
    return *this;
}
```

Think about: why does this handle both copy and move? What are the trade-offs vs separate operators?

<br>

## Challenge 3: Add a move-aware `append` method

```cpp
void append(String&& suffix);        // steal from suffix
void append(const String& suffix);   // copy from suffix
```

<br>

## Challenge 4: Track allocation counts

Add static counters:

```cpp
static int alloc_count;
static int dealloc_count;
```

Print a summary at the end to verify no leaks. Compare counts with vs without `std::move`.

<br><br>

---
---

# PART 4: REFLECT & NOTES (10 min)

---

<br>

Write answers to these in your own notes:

1. **What happens if you forget the move constructor?**
   - The class falls back to copy when receiving rvalues. No compiler error — just slower.

2. **What's the danger of using an object after `std::move`?**
   - It's in a valid-but-unspecified state. Reading it is undefined behavior *in spirit* (technically allowed, but meaningless).

3. **Why should move operations be `noexcept`?**
   - `std::vector::push_back` provides strong exception guarantee. It will only use move if it's `noexcept`; otherwise it copies (safe rollback).

4. **Copy-and-swap: what's the trade-off?**
   - Pro: single operator handles both, exception-safe by construction.
   - Con: always makes a full copy even for self-assignment; may be slightly slower than a hand-tuned copy-assign.

5. **When does the compiler elide moves entirely (RVO/NRVO)?**
   - Returning a local variable by value → compiler may construct directly in the caller's space, skipping both copy and move.

<br><br>

---
---

# INTERVIEW QUESTIONS TO PRACTICE

---

<br>

1. "Explain the difference between `std::move` and actually moving an object."
2. "What is the Rule of 5? When does it apply?"
3. "Why would declaring a destructor break your class's move semantics?"
4. "Implement a move-aware resource wrapper for a file descriptor."
5. "What happens when `std::vector` resizes and your type's move constructor is not `noexcept`?"

<br><br>

---
---

# REFERENCE SOLUTION

---

<br>

<details>
<summary>Click to expand the complete String implementation</summary>

<br>

```cpp
#include <iostream>
#include <cstring>
#include <utility>
#include <vector>

class String {
private:
    char* data_;
    size_t size_;

public:
    String(const char* str = "") {
        size_ = std::strlen(str);
        data_ = new char[size_ + 1];
        std::memcpy(data_, str, size_ + 1);
        std::cout << "  [ctor] \"" << data_ << "\"\n";
    }

    ~String() {
        std::cout << "  [dtor] \"" << (data_ ? data_ : "(null)") << "\"\n";
        delete[] data_;
    }

    String(const String& other)
        : data_(new char[other.size_ + 1])
        , size_(other.size_)
    {
        std::memcpy(data_, other.data_, size_ + 1);
        std::cout << "  [copy ctor] \"" << data_ << "\"\n";
    }

    String(String&& other) noexcept
        : data_(other.data_)
        , size_(other.size_)
    {
        std::cout << "  [move ctor] \"" << data_ << "\"\n";
        other.data_ = nullptr;
        other.size_ = 0;
    }

    String& operator=(String other) noexcept {
        std::cout << "  [assign (copy-and-swap)]\n";
        swap(*this, other);
        return *this;
    }

    friend void swap(String& a, String& b) noexcept {
        using std::swap;
        swap(a.data_, b.data_);
        swap(a.size_, b.size_);
    }

    const char* c_str() const { return data_ ? data_ : "(null)"; }
    size_t size() const { return size_; }

    friend String operator+(const String& lhs, const String& rhs) {
        char* buf = new char[lhs.size_ + rhs.size_ + 1];
        std::memcpy(buf, lhs.data_, lhs.size_);
        std::memcpy(buf + lhs.size_, rhs.data_, rhs.size_ + 1);
        String result;
        delete[] result.data_;
        result.data_ = buf;
        result.size_ = lhs.size_ + rhs.size_;
        return result;
    }
};

int main() {
    std::cout << "=== 1. Construction ===\n";
    String a("Hello");

    std::cout << "\n=== 2. Copy construction ===\n";
    String b = a;

    std::cout << "\n=== 3. Copy assignment ===\n";
    String c("World");
    c = a;

    std::cout << "\n=== 4. Move construction ===\n";
    String d = std::move(a);

    std::cout << "\n=== 5. Move assignment ===\n";
    String e("Temp");
    e = std::move(b);

    std::cout << "\n=== 6. Concatenation ===\n";
    String f = d + c;

    std::cout << "\n=== 7. Vector push_back (verifies noexcept move) ===\n";
    {
        std::vector<String> vec;
        vec.reserve(1);
        vec.push_back(String("first"));
        std::cout << "  -- triggering reallocation --\n";
        vec.push_back(String("second"));
    }

    std::cout << "\n=== Destruction ===\n";
    return 0;
}
```

</details>

<br>

---

**Next**: [Day 2 — `unique_ptr` Deep Dive →](day02-unique-ptr.md)  *(coming soon)*
