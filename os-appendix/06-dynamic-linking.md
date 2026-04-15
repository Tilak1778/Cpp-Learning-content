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

## Minix 3 Deep Dive — Program Loading and exec()

While Minix 3 uses a standard ELF loader for dynamic linking (similar to Linux's `ld.so`), the **exec() flow** — where the OS actually loads a new program into a process — is uniquely instructive because it clearly shows the three-server cooperation.

<br>

<h3 style="color: #E67E22;">Source Files for Program Loading</h3>

```
minix/servers/pm/
├── exec.c          ← PM-side exec: coordinate the overall flow

minix/servers/vfs/
├── exec.c          ← VFS-side exec: read ELF headers, load segments from file

minix/servers/vm/
├── exec.c          ← VM-side exec: destroy old address space, create new one
├── region.c        ← set up text, data, bss, stack regions

minix/lib/libc/sys/
├── execve.c        ← libc wrapper (packs args into message)

minix/include/
├── minix/exec.h    ← shared structures between PM, VFS, VM
```

<br>

<h3 style="color: #E67E22;">The exec() Message Flow in Minix 3</h3>

```
User Process           PM              VFS             VM              Kernel
     │                  │               │               │                │
     │─exec("/bin/ls")─►│               │               │                │
     │                  │               │               │                │
     │                  │──EXEC_OPEN───►│               │                │
     │                  │               │ open file     │                │
     │                  │               │ read ELF hdr  │                │
     │                  │               │ parse PHDR:   │                │
     │                  │               │  text_addr,   │                │
     │                  │               │  text_size,   │                │
     │                  │               │  data_addr,   │                │
     │                  │               │  data_size,   │                │
     │                  │               │  entry_point  │                │
     │                  │◄──header info─│               │                │
     │                  │               │               │                │
     │                  │──EXEC_NEWMEM──────────────────►│               │
     │                  │               │               │ free old       │
     │                  │               │               │ address space  │
     │                  │               │               │ create new:    │
     │                  │               │               │  text region   │
     │                  │               │               │  data region   │
     │                  │               │               │  bss region    │
     │                  │               │               │  stack region  │
     │                  │◄─────────────────────────OK───│               │
     │                  │               │               │                │
     │                  │──EXEC_LOAD───►│               │                │
     │                  │               │ read segments │                │
     │                  │               │ from file     │                │
     │                  │               │ copy into new │                │
     │                  │               │ address space │                │
     │                  │◄──────────OK──│               │                │
     │                  │               │               │                │
     │                  │──SYS_EXEC─────────────────────────────────────►│
     │                  │               │               │    set PC to   │
     │                  │               │               │    entry_point │
     │                  │               │               │    set SP to   │
     │                  │               │               │    stack top   │
     │                  │◄──────────────────────────────────────────OK───│
     │                  │               │               │                │
     │ (now running /bin/ls at entry_point)              │                │
```

<br>

<h3 style="color: #E67E22;">ELF Header Parsing in VFS</h3>

```c
/* servers/vfs/exec.c (simplified) */
int do_exec(void) {
    char path[PATH_MAX];
    /* ... get pathname from message ... */

    /* Open the executable */
    struct filp *f = open_file(path, O_RDONLY);

    /* Read the ELF header */
    Elf32_Ehdr ehdr;
    read_file(f, &ehdr, sizeof(ehdr), 0);

    /* Verify it's a valid ELF */
    if (memcmp(ehdr.e_ident, ELFMAG, 4) != 0)
        return ENOEXEC;

    /* Read program headers — these describe the segments to load */
    Elf32_Phdr phdr[ehdr.e_phnum];
    read_file(f, phdr, sizeof(phdr), ehdr.e_phoff);

    /* Extract segment info */
    struct exec_info execi;
    for (int i = 0; i < ehdr.e_phnum; i++) {
        if (phdr[i].p_type == PT_LOAD) {
            if (phdr[i].p_flags & PF_X) {
                /* Text (code) segment */
                execi.text_addr = phdr[i].p_vaddr;
                execi.text_size = phdr[i].p_memsz;
                execi.text_offset = phdr[i].p_offset;
            } else {
                /* Data segment */
                execi.data_addr = phdr[i].p_vaddr;
                execi.data_size = phdr[i].p_filesz;
                execi.bss_size  = phdr[i].p_memsz - phdr[i].p_filesz;
                execi.data_offset = phdr[i].p_offset;
            }
        }
        if (phdr[i].p_type == PT_INTERP) {
            /* Dynamic linker path (e.g., /usr/libexec/ld.elf_so) */
            read_file(f, execi.interp, phdr[i].p_filesz, phdr[i].p_offset);
        }
    }

    execi.entry_point = ehdr.e_entry;

    /* Send segment info to PM, which will forward to VM */
    return send_exec_info(pm_endpoint, &execi);
}
```

<br>

<h3 style="color: #E67E22;">ELF Segment Layout Explained</h3>

```
ELF File on Disk:                     Process Address Space After Loading:
┌─────────────────────┐               ┌─────────────────────────┐
│ ELF Header          │               │ Stack (grows down)      │
│  e_entry = 0x400080 │               │         ↓               │
│  e_phoff = 52       │               ├─────────────────────────┤
├─────────────────────┤               │ (gap — unmapped)        │
│ Program Headers     │               ├─────────────────────────┤
│  LOAD: text segment │               │ Heap (grows up)         │
│    vaddr = 0x400000 │               │         ↑               │
│    filesz = 0x1000  │               ├─────────────────────────┤
│    flags = R+X      │               │ .bss  (zeroed)  [RW]    │
│  LOAD: data segment │               │  0x601100 - 0x602000    │
│    vaddr = 0x601000 │               ├─────────────────────────┤
│    filesz = 0x100   │               │ .data (initialized)[RW] │
│    memsz = 0x1000   │               │  0x601000 - 0x601100    │
│    flags = R+W      │               ├─────────────────────────┤
│  INTERP: ld.so path │               │ .text (code)    [RX]    │
├─────────────────────┤               │  0x400000 - 0x401000    │
│ .text section       │               │  entry: 0x400080        │
│ (machine code)      │               └─────────────────────────┘
├─────────────────────┤
│ .data section       │               VM server creates 4 regions:
│ (initialized data)  │                 1. text:  addr=0x400000, len=0x1000, RX
└─────────────────────┘                 2. data:  addr=0x601000, len=0x100, RW
                                        3. bss:   addr=0x601100, len=0xF00, RW (zeroed)
                                        4. stack: addr=0x7FFFF000, len=0x10000, RW
```

<br>

<h3 style="color: #E67E22;">VM Server — Creating the New Address Space</h3>

```c
/* servers/vm/exec.c (simplified) */
int do_exec_newmem(message *msg) {
    struct vmproc *vmp = find_proc(msg->VMEN_ENDPOINT);
    struct exec_info *execi = &msg->VMEN_EXEC_INFO;

    /* Destroy old address space completely */
    free_proc_memory(vmp);

    /* Create new regions for the new program */

    /* Text (code) segment — read-only, executable */
    map_page_region(vmp, execi->text_addr, execi->text_size,
                    VR_DIRECT, &mem_type_file);

    /* Data segment — read-write, from file */
    map_page_region(vmp, execi->data_addr, execi->data_size,
                    VR_WRITABLE | VR_DIRECT, &mem_type_file);

    /* BSS segment — read-write, zeroed (anonymous) */
    map_page_region(vmp, execi->data_addr + execi->data_size,
                    execi->bss_size,
                    VR_WRITABLE, &mem_type_anon);

    /* Stack — read-write, at top of address space */
    map_page_region(vmp, STACK_BASE - DEFAULT_STACK_SIZE,
                    DEFAULT_STACK_SIZE,
                    VR_WRITABLE, &mem_type_anon);

    /* Build new page table */
    pt_new(&vmp->vm_pt);

    return OK;
}
```

<br>

<h3 style="color: #E67E22;">Dynamic Linking in Minix 3</h3>

Minix 3 uses NetBSD's dynamic linker (`ld.elf_so`). The flow after `exec()` sets up the address space:

```
1. Kernel starts the process at the DYNAMIC LINKER's entry point
   (not the program's main — the PT_INTERP header told it where ld.so is)

2. ld.elf_so runs:
   a. Maps itself into the address space
   b. Reads the executable's .dynamic section
   c. Finds needed shared libraries (LD_LIBRARY_PATH, /usr/lib)
   d. mmap's each .so (sends mmap messages to VM server)
   e. Resolves symbols, sets up PLT/GOT
   f. Calls .init constructors (global C++ objects)
   g. Jumps to the executable's actual entry point

3. main() runs
```

This is identical to Linux's flow, but each `mmap()` call goes through the VM server as a message rather than being a direct kernel call.

<br>

<h3 style="color: #E67E22;">Static vs Dynamic Linking in Minix 3 — Performance Implications</h3>

| | Static | Dynamic (Minix 3) | Dynamic (Linux) |
|---|--------|-------------------|-----------------|
| exec() syscalls | PM + VM + VFS | PM + VM + VFS + many mmap messages | Kernel-only (fewer context switches) |
| Library loading | None (all in binary) | ld.elf_so sends mmap messages to VM server | ld.so makes mmap syscalls directly |
| PLT/GOT overhead | None | Same as Linux | Same |
| Startup time | Fastest | Slowest (message passing overhead) | Middle ground |

For Minix 3, static linking is more common in system servers (PM, VFS, VM) because they need to start before the dynamic linker infrastructure is available.

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
9. "Walk through the ELF loading process during exec()."
10. "What are the PT_LOAD and PT_INTERP program headers in an ELF file?"

---

**Next**: [File System Internals →](07-file-system.md)
