---
layout: page
title: "xv6 Kernel Reference — bio.c · fs.c · file.c"
description: "Function reference, struct diagrams, and call flows for the xv6 file system kernel files."
---

# xv6 Kernel File Reference

> Complete reference for `bio.c`, `fs.c`, and `file.c` — the three core kernel files for the xv6 file system.

---

## The 5-Layer Stack

Every `read()` / `write()` syscall passes through all 5 layers in order:

```
┌─────────────────────────────────────────────────────────┐
│  Layer 5 — Syscall Interface       sysfile.c            │
│  sys_open, sys_read, sys_write, sys_close               │
├─────────────────────────────────────────────────────────┤
│  Layer 4 — File Descriptor         file.c   ← YOU ARE HERE
│  struct file, ftable, offset tracking                   │
├─────────────────────────────────────────────────────────┤
│  Layer 3 — Inode / Dir / Path      fs.c     ← YOU ARE HERE
│  ialloc, ilock, readi, writei, namei                    │
├─────────────────────────────────────────────────────────┤
│  Layer 2 — Crash Recovery          log.c                │
│  begin_op, log_write, end_op                            │
├─────────────────────────────────────────────────────────┤
│  Layer 1 — Buffer Cache            bio.c    ← YOU ARE HERE
│  bread, brelse, bwrite, LRU list                        │
├─────────────────────────────────────────────────────────┤
│  Layer 0 — Physical Disk           virtio_disk.c        │
└─────────────────────────────────────────────────────────┘
```

---

## bio.c — Buffer Cache

### What it does

`bio.c` is **Layer 1**. It sits directly above the disk driver and does two things:

1. **Caches** recently-used disk blocks in RAM so we don't re-read the same block from disk every time
2. **Synchronises** access — only one process can hold a buffer at a time (via sleeplock)

### Key Structs

**`struct bcache`** — global, one per system

| Field | Type | Purpose |
|-------|------|---------|
| `lock` | spinlock | Guards the entire LRU list |
| `buf[NBUF]` | buf array | 30 buffer slots |
| `head` | buf | Sentinel node — not a real buffer |

**`struct buf`** — one per buffer slot

| Field | Type | Purpose |
|-------|------|---------|
| `valid` | int | 1 = data was read from disk |
| `disk` | int | 1 = disk I/O in progress |
| `dev` | uint | Which device this block belongs to |
| `blockno` | uint | Which disk block number |
| `lock` | sleeplock | Per-buffer lock — only one user at a time |
| `refcnt` | uint | How many callers are using this buffer |
| `data[BSIZE]` | uchar | 1024 bytes of actual block content |

### LRU List Structure

```
head.next = Most Recently Used
head.prev = Least Recently Used (evicted first)

  HEAD <-> [blk 267, refcnt=1] <-> [blk 45, refcnt=1] <-> [blk 12, refcnt=0] <-> [blk 3, refcnt=0] <-> HEAD
  (sentinel)   LOCKED, in use        LOCKED, in use          free, can evict      LRU, evicted first
```

- `refcnt > 0` → buffer is in use, **cannot evict**
- `refcnt == 0` → buffer is free, **can evict** (LRU end gets evicted first)
- After `brelse()`, buffer moves to the **head** (MRU position)

---

### bio.c Functions

#### `binit()` — Initialise the buffer cache

```c
void binit(void)
```

**Purpose:** Called once at boot. Initialises the `bcache` spinlock and wires all 30 buffer structs into a circular doubly-linked list through the `bcache.head` sentinel. Also initialises the sleeplock inside each buffer.

**When called:** `main()` at boot, before any file system I/O.

---

#### `bread()` — Read a disk block ⭐ Most used

```c
struct buf* bread(uint dev, uint blockno)
```

**Purpose:** The main way to load a disk block into memory. Calls `bget()` to find or allocate a buffer slot, then reads the block from disk if it wasn't already cached (`valid == 0`). Returns a **locked** buffer.

| Parameter | Description |
|-----------|-------------|
| `dev` | Device number (e.g. `ROOTDEV = 1`) |
| `blockno` | Disk block number to load |

**Returns:** A locked `struct buf*` with `buf->data` containing 1024 bytes of block content.

> **⚠️ Rule:** You MUST call `brelse(bp)` after every `bread()`. Forgetting this causes the buffer cache to fill up (panic: `"bget: no buffers"`).

**Usage pattern:**
```c
struct buf *bp = bread(dev, 45);    // load block 45
uint val = *(uint*)bp->data;         // read from it
brelse(bp);                          // ALWAYS release!
```

---

#### `bwrite()` — Write buffer to disk

```c
void bwrite(struct buf *b)
```

**Purpose:** Writes the buffer's data directly to disk via `virtio_disk_rw(b, 1)`. Caller must hold the buffer's sleeplock.

> **⚠️ Do not call this directly.** In normal FS code, use `log_write(bp)` instead — it goes through the transaction log and is crash-safe. `bwrite()` bypasses crash recovery.

---

#### `brelse()` — Release a buffer ⭐ Must always call after bread

```c
void brelse(struct buf *b)
```

**Purpose:** Releases the buffer's sleeplock and decrements `refcnt`. If `refcnt` drops to 0, moves the buffer to the **head** of the LRU list (most-recently-used position).

**What happens inside:**
```c
releasesleep(&b->lock);     // step 1: unlock
b->refcnt--;                // step 2: decrement
if (b->refcnt == 0) {
    // step 3: move to MRU head of list
    b->next->prev = b->prev;
    b->prev->next = b->next;
    b->next = bcache.head.next;
    bcache.head.next = b;
}
```

---

#### `bget()` — Find or allocate a buffer slot (internal)

```c
static struct buf* bget(uint dev, uint blockno)
```

**Purpose:** Internal helper called by `bread()`. Does two passes:

1. **Pass 1 (cache hit):** Scan list front-to-back — if a buffer for this `(dev, blockno)` already exists, increment `refcnt` and return it
2. **Pass 2 (cache miss):** Scan list back-to-front (LRU end) — find a buffer with `refcnt == 0` and recycle it for this block

> **⚠️ Panic condition:** If all 30 buffers have `refcnt > 0`, panics with `"bget: no buffers"`. This means you forgot `brelse()` somewhere.

---

#### `bpin()` / `bunpin()` — Pin a buffer in cache

```c
void bpin(struct buf *b)
void bunpin(struct buf *b)
```

**Purpose:** Used by the **log layer** to keep a buffer alive in the cache during a transaction. `bpin()` increments `refcnt` without acquiring the sleeplock (prevents LRU eviction). `bunpin()` decrements it.

---

### bio.c Quick Reference

| Function | Returns | Purpose |
|----------|---------|---------|
| `binit()` | void | Boot-time init of buffer cache |
| `bread(dev, blockno)` | `*buf` (locked) | Load disk block into cache |
| `bwrite(b)` | void | Write buffer to disk directly (use `log_write` instead) |
| `brelse(b)` | void | Release buffer lock, move to MRU |
| `bget(dev, blockno)` | `*buf` (internal) | Find or recycle a buffer slot |
| `bpin(b)` | void | Pin buffer in cache (for log layer) |
| `bunpin(b)` | void | Unpin buffer |

---

---

## fs.c — File System Core

### What it does

`fs.c` is the largest file — it implements **Layers 2–5** of the file system:

- **Block layer** — allocate/free disk blocks (`balloc`, `bfree`)
- **Inode layer** — manage file metadata (`ialloc`, `ilock`, `iput`, etc.)
- **Data layer** — read/write file content (`readi`, `writei`, `bmap`)
- **Directory layer** — directory entry lookup/creation (`dirlookup`, `dirlink`)
- **Path layer** — resolve `/usr/bin/sh` to an inode (`namei`, `nameiparent`)

### On-Disk Layout (After Lab 3 — FSSIZE = 200,000)

```
Block 0       Block 1       Blocks 2-31     Blocks 32-44    Blocks 45-69    Blocks 70-199999
┌───────────┬─────────────┬──────────────┬───────────────┬───────────────┬──────────────────┐
│   Boot    │  Superblock │     Log      │    Inodes     │    Bitmap     │    Data Blocks   │
│           │             │  (30 blocks) │  (13 blocks)  │  (25 blocks)  │  (199,930 blocks)│
│bootloader │  fs metadata│crash recovery│ 200 inodes    │ free-space    │ actual file data │
│           │             │  begin_op    │ 16 per block  │ 1 bit/block   │                  │
└───────────┴─────────────┴──────────────┴───────────────┴───────────────┴──────────────────┘
```

**nmeta = 70 blocks** (1 + 1 + 30 + 13 + 25)

---

### Inode Structure

**`struct dinode`** (on disk, 64 bytes) vs **`struct inode`** (in RAM)

| Field | Type | Purpose |
|-------|------|---------|
| `type` | short | `T_FILE=2`, `T_DIR=1`, `T_DEVICE=3`, `T_SYMLINK=4` |
| `nlink` | short | Number of hard links to this inode |
| `size` | uint | File size in bytes |
| `addrs[13]` | uint[] | Block addresses — the key field |

### `addrs[]` Block Addressing (After Lab 3)

```
addrs[0]      --> Data Block 0                    (direct)
addrs[1..10]  --> Data Blocks 1-10                (direct, 11 total)
                  ─────────────────────────────────────────────
addrs[11]     --> [ Map Block ]                   (singly-indirect)
                      |
                      +--> Data Block 11
                      +--> Data Block 12
                      +--> ...
                      +--> Data Block 266          (256 blocks via 1 map)
                  ─────────────────────────────────────────────
addrs[12]     --> [ Master Map Block ]            (doubly-indirect)
                      |
                      +--> [ Secondary Map 0 ] --> Data Blocks 267-522
                      +--> [ Secondary Map 1 ] --> Data Blocks 523-778
                      +--> ...
                      +--> [ Secondary Map 255]--> Data Blocks up to 65802
                                                   (65,536 blocks via 256x256)
                  ─────────────────────────────────────────────
TOTAL MAX FILE SIZE = 11 + 256 + 65536 = 65,803 blocks (~64 MB)
```

---

### Block Layer Functions

#### `balloc()` — Allocate a free disk block

```c
static uint balloc(uint dev)
```

**Purpose:** Scans the free-space bitmap block by block. Finds the first bit = 0 (free), sets it to 1, calls `log_write()` on the bitmap block, then `bzero()` to zero-out the newly allocated block.

**Returns:** Block number on success. Returns `0` (prints `"balloc: out of blocks"`) if disk is full.

> **Called by:** `bmap()` — only when a file grows into a block that doesn't exist yet.

---

#### `bfree()` — Free a disk block

```c
static void bfree(int dev, uint b)
```

**Purpose:** Reads the bitmap block containing bit `b`, clears it to 0, calls `log_write()`. The block is now available for reuse.

> **Called by:** `itrunc()` — when deleting a file, loops over all its blocks calling `bfree()` on each.

---

### Inode Layer Functions

#### Inode Lifecycle

```
ialloc()         iget()          ilock()           iunlock()         iput()
    |               |               |                   |               |
  CREATE         GET HANDLE       LOCK + LOAD        UNLOCK         DROP REF
  (on disk)      (in memory,      (reads from         (release       (free if
  type != 0      ref++)           disk if             sleeplock)     last ref
                                  valid==0)                          && nlink==0)
```

---

#### `ialloc()` — Create a new inode

```c
struct inode* ialloc(uint dev, short type)
```

**Purpose:** Scans on-disk inodes to find one with `type == 0` (free). Sets its type, writes it back via `log_write()`, then calls `iget()` to return an in-memory handle.

| Parameter | Description |
|-----------|-------------|
| `dev` | Device to allocate on |
| `type` | `T_FILE`, `T_DIR`, `T_DEVICE`, or `T_SYMLINK` |

---

#### `iget()` — Get in-memory inode handle

```c
static struct inode* iget(uint dev, uint inum)
```

**Purpose:** Finds or creates an in-memory inode entry for `(dev, inum)`. Increments `ref` count. Does **not** read from disk or lock — just reserves the slot. Returned inode may have `valid = 0`.

> **⚠️ Important:** Does NOT lock. You must call `ilock(ip)` before reading `ip->type`, `ip->size`, or `ip->addrs[]`.

---

#### `ilock()` — Lock inode and load from disk

```c
void ilock(struct inode *ip)
```

**Purpose:** Acquires the inode's sleeplock. If `ip->valid == 0`, reads the on-disk `dinode` and populates all fields (`type`, `nlink`, `size`, `addrs[]`). After `ilock()` returns, all inode fields are safe to read and modify.

**Standard inode usage pattern:**
```c
struct inode *ip = namei("/myfile");   // step 1: get handle
ilock(ip);                             // step 2: lock + load from disk

// step 3: use ip->type, ip->size, readi(), writei()...
int r = readi(ip, 1, dst, 0, n);

iunlock(ip);                           // step 4: unlock
iput(ip);                              // step 5: drop reference
```

---

#### `iunlock()` — Unlock inode

```c
void iunlock(struct inode *ip)
```

**Purpose:** Releases the inode's sleeplock. Inode stays in memory (`ref` count unchanged). Other processes waiting on this inode can now proceed.

---

#### `iput()` — Drop inode reference

```c
void iput(struct inode *ip)
```

**Purpose:** Decrements `ref` count. If this was the **last reference** (`ref == 1`) AND the file has **no hard links** (`nlink == 0`), the file is actually deleted: calls `itrunc()` to free all data blocks, sets `type = 0`, calls `iupdate()`.

> **Shorthand:** `iunlockput(ip)` calls `iunlock(ip)` then `iput(ip)` — use this to avoid forgetting one of them.

---

#### `iupdate()` — Write inode back to disk

```c
void iupdate(struct inode *ip)
```

**Purpose:** Copies the in-memory inode fields back to the on-disk `dinode` via `bread()` → `memmove()` → `log_write()` → `brelse()`. Call this after modifying `ip->size` or `ip->nlink`.

---

#### `itrunc()` — Free all data blocks of a file

```c
void itrunc(struct inode *ip)
```

**Purpose:** Frees all disk blocks belonging to the inode — direct blocks, singly-indirect blocks, and doubly-indirect blocks. Sets `ip->size = 0`, zeros all `addrs[]`. Called by `iput()` when a file is deleted.

**Tree walk order (must release before free):**
```c
// For each doubly-indirect secondary map:
bp2 = bread(dev, a1[i]);          // load secondary map
for(j = 0; j < NINDIRECT; j++)
    if(a2[j]) bfree(dev, a2[j]); // free data blocks
brelse(bp2);                       // release BEFORE bfree
bfree(dev, a1[i]);                 // free secondary map block
```

---

#### `bmap()` — Logical block → Physical block ⭐

```c
static uint bmap(struct inode *ip, uint bn)
```

**Purpose:** The block navigator. Converts logical block number `bn` (0 = first block of file) to a physical disk block number. Handles all 3 addressing levels. Calls `balloc()` to create new blocks if they don't exist yet.

| `bn` range | Addressing level | How it navigates |
|------------|-----------------|-----------------|
| 0 – 10 | Direct | Returns `addrs[bn]` directly |
| 11 – 266 | Singly-indirect | `bread(addrs[11])` → look up `a[bn-11]` |
| 267 – 65802 | Doubly-indirect | `bread(addrs[12])` → `a[idx1]` → `bread` → `a[idx2]` |

**Index calculation for doubly-indirect:**
```c
bn -= NDIRECT;          // subtract 11
bn -= NINDIRECT;        // subtract 256
idx1 = bn / 256;        // which secondary map (0-255)
idx2 = bn % 256;        // which slot in that map (0-255)
```

> **Called by:** `readi()` and `writei()` for every block chunk of a read/write.

---

#### `readi()` — Read data from inode

```c
int readi(struct inode *ip, int user_dst, uint64 dst, uint off, uint n)
```

**Purpose:** Reads `n` bytes from inode `ip` starting at byte offset `off`. Loops block-by-block: `bmap()` → `bread()` → copy bytes out → `brelse()`.

| Parameter | Description |
|-----------|-------------|
| `user_dst` | 1 = `dst` is user virtual address; 0 = kernel address |
| `dst` | Destination buffer |
| `off` | Byte offset in file to start reading from |
| `n` | Number of bytes to read |

**Returns:** Number of bytes actually read. Returns 0 if `off >= ip->size`.

---

#### `writei()` — Write data to inode

```c
int writei(struct inode *ip, int user_src, uint64 src, uint off, uint n)
```

**Purpose:** Writes `n` bytes to inode `ip` at byte offset `off`. Like `readi()` in reverse: `bmap()` (may call `balloc()`) → `bread()` → copy data in → `log_write()` → `brelse()`. Updates `ip->size` and calls `iupdate()`.

> **⚠️ Requirements:** Must be called with `ilock(ip)` already held AND inside a `begin_op()` / `end_op()` transaction.

---

#### `dirlookup()` — Find an entry in a directory

```c
struct inode* dirlookup(struct inode *dp, char *name, uint *poff)
```

**Purpose:** Reads each `struct dirent` (16 bytes: inum + name) from directory inode `dp` using `readi()`. Compares names. If found, sets `*poff` to the byte offset of the entry and returns the inode via `iget()`.

**Returns:** Inode pointer (ref++, unlocked) if found. `0` if not found.

---

#### `dirlink()` — Add an entry to a directory

```c
int dirlink(struct inode *dp, char *name, uint inum)
```

**Purpose:** Adds a new `(name, inum)` entry to directory `dp`. First checks that `name` doesn't already exist. Finds an empty `dirent` slot and writes it using `writei()`.

**Returns:** 0 on success, -1 if name already exists or disk is full.

---

#### `namei()` — Resolve a path to its inode ⭐

```c
struct inode* namei(char *path)
struct inode* nameiparent(char *path, char *name)
```

**Purpose:** `namei()` resolves a full path like `/usr/bin/sh` to its final inode by walking each component. `nameiparent()` returns the parent directory's inode and copies the last component into `name`.

**How `/a/b/c` is resolved:**
```c
ip = iget(ROOTDEV, ROOTINO);        // start at root "/"
ilock(ip);
next = dirlookup(ip, "a", 0);      // find "a" in root
iunlockput(ip);  ip = next;

ilock(ip);
next = dirlookup(ip, "b", 0);      // find "b" in "a"
iunlockput(ip);  ip = next;

ilock(ip);
next = dirlookup(ip, "c", 0);      // find "c" in "b"
iunlockput(ip);
return next;                         // return "c"'s inode
```

---

### fs.c Quick Reference

| Function | Layer | Must hold | Purpose |
|----------|-------|-----------|---------|
| `balloc(dev)` | Block | inside begin_op | Allocate free disk block |
| `bfree(dev, b)` | Block | inside begin_op | Mark block b as free |
| `ialloc(dev, type)` | Inode | inside begin_op | Allocate new inode |
| `iget(dev, inum)` | Inode | — | Get in-memory handle, ref++ |
| `idup(ip)` | Inode | — | Increment ref (for fork/dup) |
| `ilock(ip)` | Inode | — | Lock + load from disk |
| `iunlock(ip)` | Inode | ilock held | Release sleeplock |
| `iput(ip)` | Inode | inside begin_op | Drop ref; free if last ref && nlink==0 |
| `iunlockput(ip)` | Inode | ilock held | iunlock + iput in one call |
| `iupdate(ip)` | Inode | ilock held | Write inode back to disk |
| `itrunc(ip)` | Inode | ilock held | Free all data blocks |
| `bmap(ip, bn)` | Inode | ilock held | Logical block → physical block |
| `readi(ip,...)` | Inode | ilock held | Read n bytes from offset |
| `writei(ip,...)` | Inode | ilock held + begin_op | Write n bytes at offset |
| `dirlookup(dp, name)` | Dir | ilock on dp | Find named entry, return inode |
| `dirlink(dp, name, inum)` | Dir | ilock on dp + begin_op | Add directory entry |
| `namei(path)` | Path | inside begin_op | Resolve path to inode |
| `nameiparent(path, name)` | Path | inside begin_op | Resolve to parent inode |

---

---

## file.c — File Descriptor Layer

### What it does

`file.c` is **Layer 4** — the topmost file system layer. It provides the file descriptor abstraction that user programs see. A `struct file` wraps an inode (or pipe) with a current offset and permission flags. This is what `fd = open("file", O_RDWR)` gives you.

### Key Structs

**`struct file`**

| Field | Type | Purpose |
|-------|------|---------|
| `type` | enum | `FD_NONE`, `FD_PIPE`, `FD_INODE`, `FD_DEVICE` |
| `ref` | int | Reference count (shared by fork/dup) |
| `readable` | char | 1 if opened for reading |
| `writable` | char | 1 if opened for writing |
| `pipe` | `*pipe` | Used when `type == FD_PIPE` |
| `ip` | `*inode` | Used when `type == FD_INODE` or `FD_DEVICE` |
| `off` | uint | Current read/write position (byte offset) |
| `major` | short | Device major number (for `FD_DEVICE`) |

**`struct ftable`** — global file table

| Field | Type | Purpose |
|-------|------|---------|
| `lock` | spinlock | Guards the whole table |
| `file[NFILE]` | file array | 100 file slots system-wide |

**Key insight:** A process's fd (integer 0, 1, 2…) indexes into `proc.ofile[]` which points into `ftable`. Multiple processes can share one `struct file` (same offset) via `filedup()`.

```
Process A:                    Process B (after fork):
  ofile[3] ─────────────┐      ofile[3] ─────────────┐
                         ↓                             ↓
                   ftable.file[7]  (ref=2, off=512)
                         |
                         └──> inode #23 (big.file)
```

---

### file.c Functions

#### `fileinit()` — Initialise file table

```c
void fileinit(void)
```

**Purpose:** Called once at boot. Initialises the `ftable` spinlock. Nothing else — the file slots start at `ref = 0` (free) by default.

---

#### `filealloc()` — Allocate a file struct

```c
struct file* filealloc(void)
```

**Purpose:** Scans `ftable` for a slot with `ref == 0`. Sets `ref = 1` and returns it. First step of `sys_open()` — reserves a file structure before setting its type and inode.

**Returns:** Pointer to free file struct, or `0` if all 100 slots are in use.

---

#### `filedup()` — Increment reference count

```c
struct file* filedup(struct file *f)
```

**Purpose:** Increments `f->ref`. Used by `fork()` (child inherits parent's open files) and `sys_dup()`. Both the original and duplicate fd now point to the **same** `struct file` — same offset, same position.

**Returns:** Same `f` pointer (enables `f = filedup(f)` idiom).

---

#### `fileclose()` — Close a file (decrement ref)

```c
void fileclose(struct file *f)
```

**Purpose:** Decrements `ref`. If ref reaches 0, actually closes the resource:

- `FD_PIPE` → calls `pipeclose()`
- `FD_INODE` or `FD_DEVICE` → calls `begin_op()` → `iput(ip)` → `end_op()`

`iput()` may delete the file entirely if this was the last reference AND `nlink == 0`.

> **Note:** Just having ref > 1 means the file stays open. The actual resource is only released when the **last** reference is dropped.

---

#### `filestat()` — Get file metadata

```c
int filestat(struct file *f, uint64 addr)
```

**Purpose:** Implements `fstat()`. Calls `ilock()` → `stati(ip, &st)` to fill a `struct stat` with `type`, `inum`, `nlink`, `size` → `iunlock()` → `copyout()` to write to user address space.

**Returns:** 0 on success. -1 if `type == FD_PIPE` (pipes have no stat).

---

#### `fileread()` — Read from file ⭐

```c
int fileread(struct file *f, uint64 addr, int n)
```

**Purpose:** Dispatches based on file type:

| Type | Calls |
|------|-------|
| `FD_PIPE` | `piperead()` |
| `FD_DEVICE` | device driver's read function |
| `FD_INODE` | `ilock()` + `readi(ip, 1, addr, f->off, n)` + advances `f->off` |

**FD_INODE path:**
```c
ilock(f->ip);
r = readi(f->ip, 1, addr, f->off, n);
if (r > 0)
    f->off += r;          // advance position for next read
iunlock(f->ip);
```

**Returns:** Number of bytes read. -1 if not readable.

---

#### `filewrite()` — Write to file ⭐

```c
int filewrite(struct file *f, uint64 addr, int n)
```

**Purpose:** Dispatches based on type. For `FD_INODE`, splits `n` bytes into chunks to stay within the log transaction size limit (`MAXOPBLOCKS`). Each chunk:

```c
begin_op();
ilock(f->ip);
r = writei(f->ip, 1, addr + i, f->off, n1);
if (r > 0) f->off += r;
iunlock(f->ip);
end_op();
```

> **Why chunks?** One giant `write()` would need thousands of `log_write()` calls — more than the log can hold in one transaction. Breaking into chunks keeps each transaction ≤ `MAXOPBLOCKS`.

**Returns:** Number of bytes written, or -1 on error.

---

### file.c Quick Reference

| Function | Syscall it serves | Purpose |
|----------|------------------|---------|
| `fileinit()` | boot | Init ftable spinlock |
| `filealloc()` | `open()` | Reserve a free struct file slot |
| `filedup(f)` | `dup()`, `fork()` | Share open file — increment ref |
| `fileclose(f)` | `close()` | Decrement ref; release when 0 |
| `filestat(f, addr)` | `fstat()` | Copy inode metadata to user stat |
| `fileread(f, addr, n)` | `read()` | Read n bytes; advance offset |
| `filewrite(f, addr, n)` | `write()` | Write n bytes in log-safe chunks |

---

---

## Call Flows — How the Files Connect

### Flow: `read(fd, buf, n)`

```
User: read(fd, buf, 1024)
  │
  ▼  sysfile.c
sys_read()
  → gets f = proc->ofile[fd]
  → calls fileread(f, addr, n)
  │
  ▼  file.c
fileread(f, addr, n)
  → checks f->readable
  → ilock(f->ip)
  → calls readi(f->ip, 1, addr, f->off, n)
  → f->off += r
  → iunlock(f->ip)
  │
  ▼  fs.c
readi(ip, 1, addr, off, n)
  → for each block: calls bmap(ip, off/BSIZE)
  → then bread(dev, phys_block)
  │
  ▼  fs.c
bmap(ip, logical_block)
  → checks if bn < NDIRECT → return addrs[bn]
  → else if bn < NINDIRECT → bread(addrs[11]) → a[bn-NDIRECT]
  → else → bread(addrs[12]) → a[idx1] → bread → a[idx2]
  │
  ▼  bio.c
bread(dev, phys_block)
  → bget(): check cache → if miss: virtio_disk_rw()
  → returns locked buf with data
  │
  ▼  bio.c (called by readi after copying)
brelse(bp)
  → releasesleep, refcnt--, move to MRU head
```

---

### Flow: `write(fd, buf, n)`

```
User: write(fd, buf, 1024)
  │
  ▼  sysfile.c
sys_write()
  → filewrite(f, addr, n)
  │
  ▼  file.c
filewrite() — splits into chunks
  → begin_op()            ← start transaction
  → ilock(f->ip)
  → writei(ip, 1, addr+i, f->off, n1)
  → iunlock(f->ip)
  → end_op()              ← commit transaction
  │
  ▼  fs.c
writei(ip, ...)
  → for each block: bmap(ip, off/BSIZE)  ← may call balloc()
  → bread(dev, phys_block)
  → copy data into bp->data
  → log_write(bp)         ← stage for commit (not to disk yet)
  → brelse(bp)
  → iupdate(ip)           ← write updated size back to disk
  │
  ▼  log.c (inside end_op)
commit()
  → write_log()           ← write dirty blocks to log region
  → write_head()          ← write commit record (now crash-safe)
  → install_trans()       ← copy from log to real disk locations
  → clear log header
```

---

### Flow: `symlink(target, path)` — Lab 3

```
User: symlink("/usr/bin/sh", "/mylink")
  │
  ▼  sysfile.c
sys_symlink()
  → argstr(0, target) = "/usr/bin/sh"
  → argstr(1, path)   = "/mylink"
  → begin_op()
  │
  ▼  sysfile.c
create(path, T_SYMLINK, 0, 0)
  → nameiparent("/mylink") → root inode
  → dirlookup(root, "mylink") → must return 0 (not exist)
  → ialloc(dev, T_SYMLINK)   → new inode #24
  → dirlink(root, "mylink", 24)
  → return locked inode #24
  │
  ▼  fs.c
writei(ip, 0, target, 0, strlen(target))
  → bmap(ip, 0) → balloc() → new data block
  → bread(dev, data_block)
  → copy "/usr/bin/sh" into bp->data
  → log_write(bp)
  → brelse(bp)
  → iupdate(ip)   ← saves size = 11
  │
  ▼  sysfile.c
iunlockput(ip)
  → end_op()    ← commits: new inode + dir entry + path data all atomic
```

---

### Flow: `open("link", O_RDONLY)` following a symlink — Lab 3

```
sys_open("link", O_RDONLY)
  │
  ▼
ip = namei("link")  →  T_SYMLINK inode
ilock(ip)
  │
  while (ip->type == T_SYMLINK && !(omode & O_NOFOLLOW))
  │    │
  │    ├── depth > 10?  →  iunlockput(ip); return -1   (circular link)
  │    │
  │    ├── readi(ip, 0, target, 0, MAXPATH)   ← read path from symlink's data
  │    ├── target[n] = 0                       ← null-terminate!
  │    ├── iunlockput(ip)                      ← release BEFORE namei
  │    ├── ip = namei(target)                  ← follow to new inode
  │    ├── if ip == 0: return -1               ← dangling symlink
  │    └── ilock(ip);  depth++                 ← lock new inode, loop
  │
  loop exits: ip->type != T_SYMLINK
  │
  ├── if T_DIR and not O_RDONLY → error
  ├── filealloc() → get struct file slot
  ├── fdalloc()   → assign fd number
  └── return fd
```

---

## Golden Rules

| # | Rule | Consequence if broken |
|---|------|----------------------|
| 1 | Every `bread()` must have exactly one `brelse()` | All 30 buffer slots fill up → panic `"bget: no buffers"` |
| 2 | `brelse(bp1)` BEFORE `bread(bp2)` when navigating doubly-indirect | Holding 2 buffer locks × concurrent processes = deadlock |
| 3 | Use `log_write(bp)` not `bwrite(bp)` inside transactions | Crash mid-write leaves filesystem corrupt |
| 4 | Call `ilock(ip)` before reading `ip->type`, `ip->size`, `ip->addrs[]` | `iget()` returns inode with `valid=0` — data not loaded yet |
| 5 | `iunlockput(ip)` BEFORE `namei(target)` in symlink following | Holding inode lock then acquiring another → deadlock |
| 6 | Wrap multi-step writes in `begin_op()` / `end_op()` | Partial updates on crash = corrupt filesystem |
| 7 | Call `itrunc()` to free doubly-indirect blocks too | Disk fills up after ~31 create+delete cycles (block leak) |
| 8 | `addrs[]` size must match in both `struct inode` and `struct dinode` | `memmove` in `ilock()` copies wrong number of bytes → silent corruption |
