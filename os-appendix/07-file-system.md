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

## Minix 3 Deep Dive — VFS and MFS

Minix 3's file system is perhaps the **best place to study OS file systems** because:
1. The VFS layer and concrete file systems are separate user-space servers
2. The Minix File System (MFS) is intentionally simple — based on the classic Minix FS from Tanenbaum's textbook
3. The code is small enough to read in full (~3000 lines for MFS)

<br>

<h3 style="color: #E67E22;">Source Tree — File System Components</h3>

```
minix/servers/vfs/               ← Virtual File System server
├── main.c                       ← main loop: receive message, dispatch
├── open.c                       ← open(), close(), creat()
├── read.c                       ← read(), write(), lseek()
├── path.c                       ← path resolution (name → inode)
├── link.c                       ← link(), unlink(), rename()
├── stadir.c                     ← stat(), mkdir(), rmdir()
├── mount.c                      ← mount(), umount()
├── pipe.c                       ← pipe(), FIFO support
├── device.c                     ← device file I/O (→ sends to drivers)
├── table.c                      ← syscall dispatch table
├── fproc.h                      ← per-process file state (fd table)
├── filp.h                       ← open file description (shared by fd's)
├── vnode.h                      ← VFS-level inode (abstracts over FS types)
└── request.c                    ← sends requests to concrete FS servers

minix/servers/mfs/               ← Minix File System (concrete FS)
├── main.c                       ← MFS main loop
├── table.c                      ← request dispatch
├── read.c                       ← block read/write
├── write.c                      ← write data blocks
├── open.c                       ← create/open files
├── link.c                       ← hard links, unlink
├── path.c                       ← directory lookup within MFS
├── inode.c                      ← inode management (read/write/alloc/free)
├── inode.h                      ← inode structure definition
├── super.c                      ← superblock management
├── super.h                      ← superblock structure
├── buf.h                        ← block cache buffer structure
└── cache.c                      ← block cache (MFS's equivalent of page cache)

minix/servers/ext2/              ← ext2 file system (alternative)
├── (similar structure to mfs/)
```

<br>

<h3 style="color: #E67E22;">VFS → Concrete FS Communication</h3>

This is the microkernel version of Linux's VFS dispatch. In Linux, VFS calls function pointers inside the kernel. In Minix 3, VFS **sends messages** to the file system server:

```
User calls: read(fd, buf, 100)
     │
     ▼
VFS server receives the message
  1. Look up fd in fproc[caller].fp_filp[fd]
  2. Get the filp (open file description)
  3. Get the vnode (VFS-level inode)
  4. Determine which FS server manages this file
     │
     ▼
VFS sends REQ_READ message to MFS server
  - includes: inode number, position, byte count
     │
     ▼
MFS server receives REQ_READ
  1. Look up inode in its inode cache
  2. Calculate which disk blocks contain the data
  3. Read blocks from disk (or block cache)
  4. Copy data to the requesting process's buffer
  5. Reply to VFS with bytes read
     │
     ▼
VFS receives reply, updates file position
VFS replies to user process with bytes read
```

```c
/* servers/vfs/request.c (simplified) */
int req_readwrite(endpoint_t fs_ep, ino_t inode_nr,
                  off_t pos, int rw_flag,
                  endpoint_t user_ep, char *user_buf,
                  size_t nbytes, off_t *new_pos, size_t *cum_io) {
    message m;
    m.m_type = (rw_flag == READING) ? REQ_READ : REQ_WRITE;
    m.REQ_INODE_NR = inode_nr;
    m.REQ_POSITION = pos;
    m.REQ_NBYTES = nbytes;
    m.REQ_USER_ADDR = user_buf;
    m.REQ_USER_ENDPT = user_ep;

    /* Send to the file system server and wait for reply */
    int r = fs_sendrec(fs_ep, &m);

    *new_pos = m.RES_POSITION;
    *cum_io = m.RES_NBYTES;
    return r;
}
```

<br>

<h3 style="color: #E67E22;">The MFS Inode — Minix 3's On-Disk Inode</h3>

```c
/* servers/mfs/inode.h (simplified from Minix 3) */
struct inode {
    u16_t  i_mode;            /* file type, protection bits (rwxrwxrwx) */
    u16_t  i_nlinks;          /* number of hard links to this inode */
    u16_t  i_uid;             /* owner user ID */
    u16_t  i_gid;             /* owner group ID */
    off_t  i_size;            /* file size in bytes */
    time_t i_atime;           /* last access time */
    time_t i_mtime;           /* last modification time */
    time_t i_ctime;           /* last inode change time */

    /* Block pointers — classic Minix FS uses direct + indirect */
    zone_t i_zone[NR_TZONES]; /* block pointers:
                                  [0]-[6]:  7 direct zones (blocks)
                                  [7]:      indirect zone
                                  [8]:      double indirect zone */

    /* In-memory fields (not on disk) */
    dev_t  i_dev;             /* which device this inode is on */
    ino_t  i_num;             /* inode number */
    int    i_count;           /* reference count (how many open) */
    char   i_dirt;            /* dirty flag — needs write-back */
};
```

Compare this to the ext4 inode we discussed earlier — the Minix inode is deliberately simpler:
- Only 7 direct zones (vs ext4's 12)
- Single and double indirect (vs ext4's triple indirect + extents)
- Smaller maximum file size (but much easier to understand)

<br>

<h3 style="color: #E67E22;">Block Mapping — Finding Data on Disk</h3>

```c
/* servers/mfs/read.c (simplified) */
zone_t read_map(struct inode *rip, off_t position) {
    /* Convert file position to block number */
    int block_pos = position / BLOCK_SIZE;
    zone_t zone;

    if (block_pos < NR_DZONES) {
        /* Direct zone — stored right in the inode */
        zone = rip->i_zone[block_pos];
    }
    else if (block_pos < NR_DZONES + NR_INDIRECTS) {
        /* Single indirect: read the indirect block, then index into it */
        int index = block_pos - NR_DZONES;
        zone_t *indirect_block = read_block(rip->i_zone[NR_DZONES]);
        zone = indirect_block[index];
    }
    else {
        /* Double indirect: two levels of indirection */
        int index = block_pos - NR_DZONES - NR_INDIRECTS;
        int indirect1 = index / NR_INDIRECTS;
        int indirect2 = index % NR_INDIRECTS;

        zone_t *dbl_indirect = read_block(rip->i_zone[NR_DZONES + 1]);
        zone_t *indirect     = read_block(dbl_indirect[indirect1]);
        zone = indirect[indirect2];
    }

    return zone;  /* disk block number where data lives */
}
```

```
File position → Block mapping:

Small file (< 7 blocks = 7KB with 1K blocks):
  inode.i_zone[0..6] → directly to data blocks

Medium file (< 7 + 1024 blocks):
  inode.i_zone[7] → indirect block → 1024 data block pointers

Large file (up to ~1GB):
  inode.i_zone[8] → double indirect → 1024 indirect blocks → 1024² data blocks

  ┌─────────────┐
  │    Inode     │
  │  zone[0] ───┼──► Data Block 0
  │  zone[1] ───┼──► Data Block 1
  │  ...         │
  │  zone[6] ───┼──► Data Block 6
  │  zone[7] ───┼──► ┌──────────────────┐
  │             │    │ Indirect Block   │
  │             │    │  [0] ──► Data 7  │
  │             │    │  [1] ──► Data 8  │
  │             │    │  ...             │
  │             │    │  [1023]──► Data  │
  │             │    └──────────────────┘
  │  zone[8] ───┼──► Double Indirect...
  └─────────────┘
```

<br>

<h3 style="color: #E67E22;">Directory Lookup in MFS</h3>

```c
/* servers/mfs/path.c (simplified) */
int search_dir(struct inode *dir_inode, char *name,
               ino_t *result_inode, int operation) {
    /* Read directory data block by block */
    off_t pos = 0;
    struct direct entry;  /* { ino_t d_ino; char d_name[NAME_MAX]; } */

    while (pos < dir_inode->i_size) {
        /* Read one directory entry */
        zone_t block = read_map(dir_inode, pos);
        read_block_data(block, pos % BLOCK_SIZE, &entry, sizeof(entry));

        if (operation == LOOK_UP) {
            if (entry.d_ino != 0 && strcmp(entry.d_name, name) == 0) {
                *result_inode = entry.d_ino;
                return OK;  /* Found it! */
            }
        }
        else if (operation == DELETE) {
            if (entry.d_ino != 0 && strcmp(entry.d_name, name) == 0) {
                entry.d_ino = 0;  /* mark entry as free */
                write_block_data(block, pos % BLOCK_SIZE,
                                 &entry, sizeof(entry));
                return OK;
            }
        }
        else if (operation == CREATE) {
            if (entry.d_ino == 0) {
                /* Found empty slot — fill it */
                entry.d_ino = *result_inode;
                strcpy(entry.d_name, name);
                write_block_data(block, pos % BLOCK_SIZE,
                                 &entry, sizeof(entry));
                return OK;
            }
        }

        pos += sizeof(entry);
    }

    return ENOENT;  /* not found */
}
```

This is a linear scan — simple but slow for directories with many entries. Linux's ext4 uses HTree (hash tree) indexing for O(1) lookup. The simplicity of Minix's approach makes it perfect for understanding the fundamentals before studying production optimizations.

<br>

<h3 style="color: #E67E22;">The MFS Block Cache</h3>

MFS maintains its own block cache (similar to Linux's page cache, but at the file system level):

```c
/* servers/mfs/cache.c (simplified) */
struct buf {
    block_t   b_blocknr;    /* which disk block */
    dev_t     b_dev;        /* which device */
    char      b_data[BLOCK_SIZE];  /* cached block data */
    char      b_dirt;       /* dirty flag — needs write-back */
    int       b_count;      /* reference count */
    struct buf *b_next;     /* hash chain */
    struct buf *b_prev;     /* LRU chain */
};

struct buf *get_block(dev_t dev, block_t block, int how) {
    /* Search hash table for cached block */
    struct buf *bp = search_hash(dev, block);

    if (bp) {
        bp->b_count++;
        return bp;  /* cache hit — fast! */
    }

    /* Cache miss — find an unused buffer (LRU eviction) */
    bp = lru_evict();

    if (bp->b_dirt) {
        /* Evicted buffer is dirty — write it back to disk first */
        write_block_to_disk(bp);
    }

    /* Read the requested block from disk */
    bp->b_blocknr = block;
    bp->b_dev = dev;
    if (how == NORMAL) {
        read_block_from_disk(bp);
    }

    return bp;
}

void put_block(struct buf *bp) {
    bp->b_count--;
    if (bp->b_count == 0) {
        /* Move to tail of LRU list (least recently used) */
        lru_append(bp);
    }
}
```

<br>

<h3 style="color: #E67E22;">The Superblock — File System Metadata</h3>

```c
/* servers/mfs/super.h (simplified) */
struct super_block {
    u32_t  s_ninodes;       /* total number of inodes */
    u16_t  s_nzones;        /* total number of zones (blocks) */
    u16_t  s_imap_blocks;   /* number of inode bitmap blocks */
    u16_t  s_zmap_blocks;   /* number of zone bitmap blocks */
    u16_t  s_firstdatazone; /* block number of first data zone */
    u16_t  s_log_zone_size; /* log2 of blocks per zone */
    off_t  s_max_size;      /* maximum file size */
    u16_t  s_magic;         /* magic number (identifies FS type) */
    /* ... */
};

/* On-disk layout:
   Block 0:  Boot block (for bootloader, even if unused)
   Block 1:  Superblock
   Blocks 2..N:  Inode bitmap (1 bit per inode: free/used)
   Blocks N..M:  Zone bitmap (1 bit per zone: free/used)
   Blocks M..K:  Inode table (array of inode structs)
   Blocks K..end: Data zones (actual file data)
*/
```

<br>

<h3 style="color: #E67E22;">Comparison: Linux ext4 vs Minix 3 MFS</h3>

| Feature | ext4 (Linux) | MFS (Minix 3) |
|---------|-------------|---------------|
| Block addressing | Extents (contiguous ranges) | Indirect block pointers |
| Directory lookup | HTree (O(1) hash) | Linear scan (O(n)) |
| Journal | Yes (crash recovery) | No (simpler, but less safe) |
| Max file size | 16 TB | ~1 GB |
| Block size | 1K–64K (typically 4K) | 1K–4K |
| Inode allocation | Flex block groups | Simple bitmap |
| Code size | ~30K lines | ~3K lines |
| Learning value | Production-grade, complex | Textbook-grade, readable |

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
9. "Walk through a read() call from user-space to disk block in Minix 3."
10. "How does the Minix FS block cache implement LRU eviction?"
11. "How does directory entry lookup work in MFS? What are its limitations?"

---

**Next**: [Pipes & IPC →](08-pipes-ipc.md)
