# OS Appendix 6: Dynamic Linking

[← Back to OS Appendix](README.md) | [← Memory-Mapped I/O](05-mmap.md)

---
---

# DYNAMIC LINKING — SHARED LIBRARIES, PLT/GOT, DLOPEN

---

<br>

<h2 style="color: #2980B9;">📘 6.1 Static vs Dynamic Linking</h2>

```
Static Linking (at compile time):
  ┌──────────┐     ┌──────────┐     ┌───────────────────────┐
  │ main.o   │ ──► │          │     │ my_program (executable)│
  │          │     │  Linker  │ ──► │ contains ALL code:     │
  │ utils.o  │ ──► │  (ld)    │     │  main + utils + libc   │
  │          │     │          │     │  (10 MB)               │
  │ libc.a   │ ──► │          │     └───────────────────────┘
  └──────────┘     └──────────┘

  ✓ Self-contained — runs anywhere
  ✗ Large binary — every program has its own copy of libc
  ✗ Bug in library → must recompile and redistribute everything

Dynamic Linking (at load time / runtime):
  ┌──────────┐     ┌──────────┐     ┌───────────────────────┐
  │ main.o   │ ──► │          │     │ my_program (executable)│
  │          │     │  Linker  │ ──► │ contains YOUR code +   │
  │ utils.o  │ ──► │  (ld)    │     │ stubs for libc         │
  └──────────┘     └──────────┘     │ (100 KB)               │
                                    └──────────┬────────────┘
                                               │ at load time
                                    ┌──────────▼────────────┐
                                    │ libc.so (shared lib)  │
                                    │ loaded into memory    │
                                    │ ONCE, shared by all   │
                                    │ processes              │
                                    └───────────────────────┘

  ✓ Small binary — library code shared across processes
  ✓ Bug fix in library → just update .so, all programs benefit
  ✗ Dependency on correct .so being present at runtime
  ✗ Slightly slower (indirection through PLT/GOT)
```

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 6.2 How a Shared Library Is Loaded</h2>

When you run a dynamically linked program:

```
$ ./my_program

1. Kernel loads the executable into memory
2. Kernel sees PT_INTERP header → "this needs a dynamic linker"
3. Kernel loads the dynamic linker: /lib64/ld-linux-x86-64.so.2
4. Dynamic linker (ld.so) takes over:
   a. Reads the executable's .dynamic section → list of needed .so files
   b. Finds each .so (search: LD_LIBRARY_PATH, /etc/ld.so.cache, /lib, /usr/lib)
   c. mmap's each .so into the process's address space (MAP_PRIVATE for code)
   d. Resolves symbols — connects function calls to actual addresses
   e. Runs constructors (__attribute__((constructor)), global C++ constructors)
5. Jumps to main()
```

```
Process address space after loading:
┌──────────────────────┐
│  Stack               │
│  ...                 │
│  libc.so    (mmap'd) │  ← shared library code + data
│  libpthread.so       │  ← another shared library
│  ...                 │
│  Heap                │
│  .data / .bss        │  ← executable's globals
│  .text               │  ← executable's code
│  ld-linux.so         │  ← the dynamic linker itself
└──────────────────────┘
```

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 6.3 Position-Independent Code (PIC)</h2>

Shared libraries must work at **any address** — different processes may load them at different locations (ASLR). This requires **position-independent code (PIC)**:

```
Without PIC (absolute addressing):
  call 0x401234    ← hardcoded address — only works if loaded at expected location

With PIC (relative addressing):
  call [rip + offset]  ← relative to current instruction pointer
  → works at ANY address
```

Compile shared libraries with `-fPIC`:
```bash
g++ -fPIC -shared -o libutils.so utils.cpp
```

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 6.4 PLT and GOT — Lazy Symbol Resolution</h2>

When your code calls a function from a shared library (e.g., `printf`), the compiler doesn't know the address at compile time. It uses two tables:

- **GOT (Global Offset Table)** — a table of pointers to actual function addresses (in the data section — writable)
- **PLT (Procedure Linkage Table)** — small stub code that jumps through the GOT (in the code section — read-only)

```
First call to printf():

Your code:          PLT:                      GOT:
call printf@PLT ──► printf@PLT:               printf@GOT:
                    │ jmp [printf@GOT] ──────► [resolver address]
                    │                          │
                    │ ◄────────────────────────┘
                    │ push index               (GOT initially points back to PLT)
                    │ jmp resolver
                    │
                    ▼
                    Dynamic linker (ld.so):
                    1. Looks up "printf" in libc.so
                    2. Finds address: 0x7f12345678
                    3. Patches GOT: printf@GOT = 0x7f12345678
                    4. Jumps to printf at 0x7f12345678

Second call to printf():

Your code:          PLT:                      GOT:
call printf@PLT ──► printf@PLT:               printf@GOT:
                    │ jmp [printf@GOT] ──────► [0x7f12345678] ──► printf() directly!
                                               (already resolved — fast!)
```

This is called **lazy binding** — symbols are resolved on first call, not at load time. This makes startup faster (not all symbols may be used).

You can force eager binding with `LD_BIND_NOW=1` or linking with `-Wl,-z,now`.

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 6.5 <code>dlopen</code> / <code>dlsym</code> — Runtime Dynamic Loading</h2>

You can load shared libraries **at runtime**, not just at load time. This enables **plugin architectures**:

```cpp
#include <dlfcn.h>
#include <iostream>

int main() {
    // Load the library at runtime
    void* handle = dlopen("./libplugin.so", RTLD_LAZY);
    if (!handle) {
        std::cerr << "dlopen failed: " << dlerror() << "\n";
        return 1;
    }

    // Look up a function by name
    using CreateFunc = void* (*)();
    auto create = (CreateFunc)dlsym(handle, "create_plugin");
    if (!create) {
        std::cerr << "dlsym failed: " << dlerror() << "\n";
        dlclose(handle);
        return 1;
    }

    // Call the function
    void* plugin = create();

    // ... use the plugin ...

    // Unload when done
    dlclose(handle);
    return 0;
}
```

Link with `-ldl`:
```bash
g++ main.cpp -ldl -o my_program
```

<br>

#### The Plugin Pattern

```cpp
// plugin_interface.h — shared between host and plugins
class IPlugin {
public:
    virtual ~IPlugin() = default;
    virtual void execute() = 0;
};

// libplugin.so — the plugin
class MyPlugin : public IPlugin {
public:
    void execute() override { std::cout << "Plugin running!\n"; }
};

extern "C" IPlugin* create_plugin() {
    return new MyPlugin();
}

// The extern "C" prevents C++ name mangling — dlsym needs the plain name
```

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 6.6 Name Mangling and <code>extern "C"</code></h2>

C++ compilers **mangle** function names to encode overloading information:

```
C++ function:        Mangled name (gcc):
void foo(int)    →   _Z3fooi
void foo(double) →   _Z3food
int bar()        →   _Z3barv
```

`dlsym` looks up symbols by their **mangled** name. Since mangling is compiler-specific, the solution is `extern "C"`:

```cpp
extern "C" void my_function();
// → symbol name is literally "my_function" — no mangling
```

Always use `extern "C"` for functions you want to find with `dlsym`.

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 6.7 ABI Compatibility</h2>

**ABI (Application Binary Interface)** = the binary-level contract between a shared library and its users. It includes:

| ABI Component | What It Means |
|---------------|---------------|
| **Symbol names** | The mangled names of functions |
| **Calling convention** | How arguments are passed (registers? stack?) |
| **Struct layout** | Size, alignment, field offsets |
| **vtable layout** | Order of virtual functions |
| **Exception handling** | Stack unwinding format |

Breaking ABI = your program crashes or behaves incorrectly when loaded with a new library version.

```
ABI break examples:
  ✗ Adding a data member to a class → changes sizeof → existing code has wrong offsets
  ✗ Reordering virtual functions → wrong vtable entry called
  ✗ Changing a function signature → stack corruption
  ✗ Removing a symbol → dlsym fails, loader can't resolve

ABI-safe changes:
  ✓ Adding a new function
  ✓ Adding a new class
  ✓ Fixing a bug without changing any signatures
  ✓ Using Pimpl idiom (Day 6) → private changes hidden from users
```

This is why Pimpl exists — it's a **compilation firewall** AND an **ABI firewall**.

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 6.8 Symbol Visibility</h2>

By default, all symbols in a shared library are exported (visible). This is usually too much — it increases load time and creates ABI surface area.

Control visibility with:

```cpp
// Export only specific symbols
__attribute__((visibility("default"))) void public_api();

// Hide everything else
// Compile with: -fvisibility=hidden
void internal_function();  // hidden — not accessible from outside the .so
```

Or use a linker version script:
```
{
    global:
        create_plugin;
        destroy_plugin;
    local:
        *;
};
```

<br><br>

---

## Interview Questions

1. "What is the difference between static and dynamic linking?"
2. "How does the PLT/GOT mechanism work? What is lazy binding?"
3. "What is position-independent code (PIC)? Why is it needed?"
4. "How would you implement a plugin system in C++?"
5. "What is ABI compatibility? How can you break it?"
6. "What is `extern \"C\"` and why is it needed with `dlsym`?"
7. "How does `dlopen`/`dlsym` work?"
8. "What is the relationship between Pimpl idiom and ABI stability?"

---

**Next**: [File System Internals →](07-file-system.md)
