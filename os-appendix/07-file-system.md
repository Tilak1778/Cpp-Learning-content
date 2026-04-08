# OS Appendix 7: File System Internals

[← Back to OS Appendix](README.md) | [← Dynamic Linking](06-dynamic-linking.md)

---
---

# FILE SYSTEM INTERNALS — INODES, BLOCKS, VFS

---

<br>

<h2 style="color: #2980B9;">📘 7.1 The File System Abstraction</h2>

To user-space programs, a file is just a stream of bytes with a name and a path. Under the hood, the file system manages:

- Where the bytes are physically stored on disk
- Metadata (size, permissions, timestamps)
- Directory structure (mapping names to files)
- Free space management

```
What you see:                    What the disk actually stores:
/home/user/notes.txt             Inode 42:
  "Hello, world!"                  size: 13
                                   permissions: rw-r--r--
                                   owner: user
                                   blocks: [block 1037]
                                 
                                 Block 1037:
                                   48 65 6C 6C 6F 2C 20 77
                                   6F 72 6C 64 21 ...
                                   ("Hello, world!")
```

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 7.2 Disk Layout (ext4 Example)</h2>

```
Disk partitioned into Block Groups:

┌───────────┬─────────────┬─────────────┬─────────────┬──────┐
│ Boot      │ Block       │ Block       │ Block       │      │
│ Sector    │ Group 0     │ Group 1     │ Group 2     │ ...  │
│ (MBR)     │             │             │             │      │
└───────────┴─────────────┴─────────────┴─────────────┴──────┘

Each Block Group:
┌──────────┬──────────┬─────────┬──────────┬──────────┬──────────┐
│ Super    │ Group    │ Block   │ Inode    │ Inode    │ Data     │
│ Block    │ Descr.   │ Bitmap  │ Bitmap   │ Table    │ Blocks   │
│          │          │         │          │          │          │
│ FS meta  │ Group    │ 1 bit   │ 1 bit    │ Array of │ Actual   │
│ data     │ info     │ per     │ per      │ inode    │ file     │
│          │          │ block:  │ inode:   │ structs  │ contents │
│          │          │ free?   │ free?    │          │          │
└──────────┴──────────┴─────────┴──────────┴──────────┴──────────┘
```

**Block size** is typically 4 KB (same as a memory page — not a coincidence).

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 7.3 Inodes — The File's Identity</h2>

Every file and directory has an **inode** — a data structure that stores all metadata EXCEPT the file name:

```
Inode structure (simplified):
┌──────────────────────────────────┐
│ Inode Number: 42                 │  ← unique ID within the filesystem
│ File Type: regular file          │
│ Permissions: rw-r--r--          │
│ Owner UID: 1000                  │
│ Group GID: 1000                  │
│ Size: 13 bytes                   │
│ Link Count: 1                    │  ← number of directory entries pointing here
│ Timestamps:                      │
│   atime: last access             │
│   mtime: last modification       │
│   ctime: last inode change       │
│ Block Pointers:                  │
│   Direct[0]: block 1037          │  ← first 12 blocks pointed to directly
│   Direct[1]: —                   │
│   ...                            │
│   Direct[11]: —                  │
│   Indirect: —                    │  ← points to a block of block pointers
│   Double Indirect: —             │
│   Triple Indirect: —             │
└──────────────────────────────────┘
```

<br>

#### Block Pointer Hierarchy (for large files):

```
For small files (< 48 KB with 4KB blocks):
  Inode → 12 direct block pointers → data blocks

For medium files (< ~4 MB):
  Inode → Indirect pointer → block of 1024 pointers → data blocks

For large files (< ~4 GB):
  Inode → Double indirect → block of pointers → blocks of pointers → data

For huge files:
  Inode → Triple indirect → ... → ... → ... → data
```

Modern file systems (ext4) use **extents** instead of this tree — an extent says "blocks 1000-2000 are contiguous" which is much more efficient.

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 7.4 Directories — Just Files That Map Names to Inodes</h2>

A directory is a file whose content is a list of `(name, inode number)` entries:

```
Directory /home/user/ (inode 100):
┌──────────────┬──────────┐
│ Name         │ Inode #  │
├──────────────┼──────────┤
│ .            │ 100      │  ← this directory itself
│ ..           │ 50       │  ← parent directory
│ notes.txt   │ 42       │
│ photos/     │ 67       │  ← subdirectory
│ readme.md   │ 42       │  ← SAME inode as notes.txt! (hard link)
└──────────────┴──────────┘
```

**Path resolution** = walking the directory tree:

```
open("/home/user/notes.txt"):
  1. Read inode 2 (root directory /)
  2. Find "home" → inode 30
  3. Read inode 30 (directory /home)
  4. Find "user" → inode 50
  5. Read inode 50 (directory /home/user)
  6. Find "notes.txt" → inode 42
  7. Read inode 42 → get file metadata and block pointers
  8. Return file descriptor
```

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 7.5 Hard Links vs Symbolic Links</h2>

```
Hard Link:
  Two directory entries pointing to the SAME inode.
  
  /home/user/notes.txt ──► inode 42 ◄── /home/user/readme.md
                            link count = 2
  
  - Deleting one entry decrements link count
  - File is actually deleted when link count reaches 0
  - Cannot cross filesystem boundaries
  - Cannot link to directories (would create cycles)

Symbolic (Soft) Link:
  A separate file whose content is a PATH to another file.
  
  /home/user/shortcut.txt (inode 99, type=symlink)
    content: "/home/user/notes.txt"
        │
        ▼
  /home/user/notes.txt (inode 42)
  
  - Is its own inode with its own metadata
  - Can cross filesystems
  - Can link to directories
  - Can be "dangling" (target deleted)
```

```bash
# Hard link
ln notes.txt readme.md        # readme.md → same inode as notes.txt
ls -i notes.txt readme.md     # same inode number

# Symbolic link
ln -s notes.txt shortcut.txt  # shortcut.txt → path "notes.txt"
ls -l shortcut.txt            # shows -> notes.txt
```

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 7.6 The VFS — Virtual File System Layer</h2>

Linux supports many file systems (ext4, XFS, Btrfs, NFS, FAT32, ...). The **VFS** provides a **uniform interface** so user code doesn't care which filesystem is underneath:

```
User space:
  open(), read(), write(), close()
       │
       ▼
┌──────────────────────────────────┐
│ VFS (Virtual File System)        │
│                                  │
│ Defines abstract operations:     │
│   inode_operations               │
│   file_operations                │
│   super_operations               │
│   dentry_operations              │
└──────────┬───────────────────────┘
           │ dispatches to concrete filesystem
    ┌──────┼──────┬──────────┐
    ▼      ▼      ▼          ▼
  ext4    XFS   Btrfs    procfs (/proc)
  driver  driver driver   (virtual — no disk)
```

This is the **Strategy Pattern** at the kernel level — each filesystem provides its own implementation of the VFS operations.

**Key VFS abstractions:**
- **superblock** — filesystem metadata (block size, total blocks, mount info)
- **inode** — file metadata (permissions, size, block pointers)
- **dentry** — directory entry cache (name → inode mapping, cached for speed)
- **file** — open file instance (position, mode, reference to dentry/inode)

```
open("/home/user/notes.txt") creates:

  struct file {
      dentry* → dentry for "notes.txt" → inode 42
      f_pos = 0  (current read/write position)
      f_mode = O_RDONLY
      f_op = ext4_file_operations  (vtable of read/write/etc)
  }
  
  fd = 3 (index into process's file descriptor table)
  fd_table[3] = pointer to this struct file
```

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 7.7 File Descriptors — The Process's View</h2>

Each process has a **file descriptor table** — an array of pointers to open file structures:

```
Process's fd table:
┌─────┬───────────────────────────────┐
│ fd  │ Points to                     │
├─────┼───────────────────────────────┤
│  0  │ → stdin  (terminal)           │
│  1  │ → stdout (terminal)           │
│  2  │ → stderr (terminal)           │
│  3  │ → notes.txt (regular file)    │
│  4  │ → socket (TCP connection)     │
│  5  │ → pipe (read end)             │
└─────┴───────────────────────────────┘

fd is just an integer index into this table.
All I/O in Unix goes through file descriptors — "everything is a file."
```

After `fork()`, the child gets a **copy** of the fd table — both parent and child share the same underlying open file descriptions (including file position).

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 7.8 The Page Cache — Why Your Disk Feels Fast</h2>

Linux caches file data in RAM using the **page cache**:

```
First read():
  Disk → Page Cache (RAM) → User buffer
         (stored for future use)

Second read() of same data:
  Page Cache (RAM) → User buffer
  (disk not touched — fast!)

write():
  User buffer → Page Cache (marked "dirty")
  → Kernel flushes dirty pages to disk periodically (pdflush/writeback)
  → Or immediately on fsync()
```

The page cache uses **all available free RAM**. This is why Linux systems show low "free" memory — it's being used for caching, which is good.

```bash
$ free -h
              total        used        free      shared  buff/cache   available
Mem:           16G         4G          500M       100M      11.5G        11G

# 11.5G is page cache — instantly reclaimable if programs need it
```

<br><br>

---

## Interview Questions

1. "What is an inode? What information does it store?"
2. "How does path resolution work? (open `/home/user/file.txt`)"
3. "What is the difference between a hard link and a symbolic link?"
4. "What is the VFS? Why does Linux need it?"
5. "What is the page cache? How does it affect read/write performance?"
6. "What happens to file descriptors after `fork()`?"
7. "Why is the file name NOT stored in the inode?"
8. "How does ext4 handle large files? (extents vs indirect blocks)"

---

**Next**: [Pipes & IPC →](08-pipes-ipc.md)
