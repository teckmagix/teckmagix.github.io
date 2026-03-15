---
layout: page
title: "Lab Predictions - lab3-m8q1"
lab: lab3
description: "Exam predictions for Lab3-w8: bmap index tracing extended, itrunc partial counts, symlink open traversal, buffer cache bread/brelse rules, struct layout correctness, and nmeta edge cases."
---

# Lab3-w8 Exam Predictions (Extended Set)

---

## Topic 1 — `bmap()` Index Tracing (More Block Numbers)

After Task 1, the three-tier layout is:

| Slot | Block range | Type |
|---|---|---|
| `addrs[0..10]` | 0 – 10 | Direct (11 blocks) |
| `addrs[11]` | 11 – 266 | Singly-indirect (256 blocks) |
| `addrs[12]` | 267 – 65802 | Doubly-indirect (256 × 256 blocks) |

**Formulas for doubly-indirect zone:**
```
bn     = logical_block - 267
index1 = bn / 256     ← which Secondary Map Book (row in Master Map)
index2 = bn % 256     ← which slot inside that Secondary Map Book
```

---

### Variation A — Logical block 267 (first doubly-indirect block)

```
bn     = 267 - 267 = 0
index1 = 0 / 256 = 0
index2 = 0 % 256 = 0
```

Path: `addrs[12]` → Master Map[0] → Secondary Map #0[0] → Data Block

> **Exam trap:** This is the very first block to enter the doubly-indirect tier. Both indices are 0 — a new Master Map AND a new Secondary Map must be allocated by `balloc()`.

---

### Variation B — Logical block 523 (first block in Secondary Map #1)

```
bn     = 523 - 267 = 256
index1 = 256 / 256 = 1
index2 = 256 % 256 = 0
```

Path: `addrs[12]` → Master Map[1] → Secondary Map #1[0] → Data Block

> **Exam trap:** Block 522 is the last block in Secondary Map #0. Block 523 is the first block in Secondary Map #1. Students who forget `index1 = bn / 256` (not `bn / 255`) get this wrong.

---

### Variation C — Logical block 779 (middle of Secondary Map #2)

```
bn     = 779 - 267 = 512
index1 = 512 / 256 = 2
index2 = 512 % 256 = 0
```

Path: `addrs[12]` → Master Map[2] → Secondary Map #2[0] → Data Block

---

### Variation D — Logical block 800

```
bn     = 800 - 267 = 533
index1 = 533 / 256 = 2
index2 = 533 % 256 = 21
```

Path: `addrs[12]` → Master Map[2] → Secondary Map #2[21] → Data Block

---

### Variation E — Logical block 522 (last block in Secondary Map #0)

```
bn     = 522 - 267 = 255
index1 = 255 / 256 = 0
index2 = 255 % 256 = 255
```

Path: `addrs[12]` → Master Map[0] → Secondary Map #0[255] → Data Block

> **Exam trap:** 255 / 256 = 0 (integer division). This is still Secondary Map #0 — it has NOT crossed the boundary to Secondary Map #1 yet. Block 523 is where that transition happens.

---

### Variation F — Logical block 33034 (middle of the doubly-indirect range)

```
bn     = 33034 - 267 = 32767
index1 = 32767 / 256 = 127
index2 = 32767 % 256 = 255
```

Path: `addrs[12]` → Master Map[127] → Secondary Map #127[255] → Data Block

---

### Variation G — Logical block 10 (direct, not indirect at all)

```
10 ≤ 10  →  falls in direct zone
```

Path: `addrs[10]` → Data Block directly

> **Exam trap:** No bread/brelse or index calculation needed. Direct blocks are a single lookup — `return ip->addrs[bn]`. Students who try to apply the doubly-indirect formula to direct blocks get completely wrong answers.

---

### Variation H — Logical block 11 (first singly-indirect block)

```
11 - 11 = 0   (bn relative to singly-indirect start)
```

Path: `addrs[11]` → Map Block[0] → Data Block

> The singly-indirect formula is simply `bn - NDIRECT`. No division or modulo needed — there is only one level of indirection.

---

## Topic 2 — `itrunc()` Block Count for Specific File Sizes

`itrunc()` frees every allocated block. The key rule: map blocks (the singly-indirect map, each secondary map, and the master map) are **separate blocks** that must each also be freed.

**bfree() call formula:**
```
Direct zone:          1 call per non-zero addrs[0..10]
Singly-indirect zone: 1 call per data block + 1 call for the map block itself
Doubly-indirect zone: 1 call per data block
                    + 1 call per Secondary Map block
                    + 1 call for the Master Map block
```

---

### Variation A — File with exactly 1 block

Only `addrs[0]` is non-zero.

```
bfree() calls = 1
```

---

### Variation B — File with exactly 11 blocks (direct zone full, no indirect)

`addrs[0..10]` all non-zero. `addrs[11]` = 0.

```
bfree() calls = 11
```

---

### Variation C — File with 12 blocks (11 direct + 1 singly-indirect data block)

`addrs[0..10]` non-zero, `addrs[11]` non-zero with 1 entry filled.

```
bfree() calls = 11 (direct)
              + 1  (one singly-indirect data block)
              + 1  (the singly-indirect map block itself)
            = 13
```

---

### Variation D — File with 267 blocks (direct full + singly-indirect full, no doubly)

All 11 direct + all 256 singly-indirect data blocks used. `addrs[12]` = 0.

```
bfree() calls = 11  (direct)
              + 256 (singly-indirect data blocks)
              + 1   (singly-indirect map block)
            = 268
```

---

### Variation E — File with 268 blocks (267 + 1 doubly-indirect data block)

Doubly-indirect zone has Master Map allocated, Secondary Map #0 allocated with 1 data block.

```
bfree() calls = 11  (direct)
              + 256 (singly-indirect data)
              + 1   (singly-indirect map)
              + 1   (one doubly-indirect data block)
              + 1   (Secondary Map #0)
              + 1   (Master Map)
            = 271
```

---

### Variation F — File with exactly 523 blocks (267 + full Secondary Map #0 + first slot of Secondary Map #1)

Doubly-indirect: Master Map allocated, Secondary Map #0 fully used (256 data blocks), Secondary Map #1 allocated with 1 data block.

```
bfree() calls = 11   (direct)
              + 257  (singly-indirect: 256 data + 1 map)
              + 256  (Secondary Map #0 data blocks)
              + 1    (one data block in Secondary Map #1)
              + 1    (Secondary Map #0 block itself)
              + 1    (Secondary Map #1 block itself)
              + 1    (Master Map)
            = 528
```

---

### Variation G — Full file (all 65,803 blocks)

```
bfree() calls = 11     (direct)
              + 256    (singly-indirect data)
              + 1      (singly-indirect map)
              + 65536  (doubly-indirect data: 256 × 256)
              + 256    (Secondary Map blocks)
              + 1      (Master Map)
            = 66,061
```

---

### Variation H — Common bug: only freeing data blocks, ignoring map blocks

A student writes `itrunc()` that frees all data blocks but never calls `bfree()` on Secondary Maps or the Master Map.

For a full file, the leak is:
- 256 Secondary Map blocks leaked
- 1 Master Map block leaked

**Total leaked: 257 blocks** — disk fills up silently as files are created and deleted over time.

---

## Topic 3 — Symlink `open()` Traversal Scenarios

When `sys_open()` encounters a file of type `T_SYMLINK` and `O_NOFOLLOW` is not set, it follows the link chain until it reaches a non-symlink file or hits the depth limit of 10.

---

### Variation A — Single hop: `link → real_file`

```
open("link", O_RDONLY)
```

Depth trace:

| Depth | Current inode | Type | Action |
|---|---|---|---|
| 1 | `link` | `T_SYMLINK` | Read path `"real_file"`, follow |
| 2 | `real_file` | `T_FILE` | Loop exits |

`open()` returns a valid file descriptor. **Success.**

---

### Variation B — Three-hop chain: `c → b → a → real_file`

```
open("c", O_RDONLY)
```

Depth trace:

| Depth | Current inode | Type | Action |
|---|---|---|---|
| 1 | `c` | `T_SYMLINK` | Read `"b"`, follow |
| 2 | `b` | `T_SYMLINK` | Read `"a"`, follow |
| 3 | `a` | `T_SYMLINK` | Read `"real_file"`, follow |
| 4 | `real_file` | `T_FILE` | Loop exits |

`open()` returns a valid file descriptor. Depth 4 is within the limit of 10. **Success.**

---

### Variation C — Circular: `a → b → a`

```
symlink("b", "a")
symlink("a", "b")
open("a", O_RDONLY)
```

Depth trace:

| Depth | Current inode | Type | Action |
|---|---|---|---|
| 1 | `a` | `T_SYMLINK` | Read `"b"`, follow |
| 2 | `b` | `T_SYMLINK` | Read `"a"`, follow |
| 3 | `a` | `T_SYMLINK` | Read `"b"`, follow |
| ... | ... | ... | ... |
| 10 | `b` | `T_SYMLINK` | Depth limit hit → return **−1** |

`open()` returns **−1**. Kernel does **not** panic or hang.

---

### Variation D — Dangling: `link → deleted_file`

```
symlink("real_file", "link")
unlink("real_file")
open("link", O_RDONLY)
```

Step-by-step:
1. `namei("link")` — finds the symlink inode. OK.
2. `ilock(ip)` — locked.
3. `readi(ip, ...)` — reads path string `"real_file"`.
4. `iunlock(ip)`, `iput(ip)` — released.
5. `namei("real_file")` — returns **0** (inode not found).
6. `sys_open()` returns **−1**.

> Kernel must **not** panic here. A null return from `namei()` is expected and handled.

---

### Variation E — `O_NOFOLLOW` set: symlink opened as a file

```
open("link", O_RDONLY | O_NOFOLLOW)
```

The `while` loop condition:
```c
while(ip->type == T_SYMLINK && !(omode & O_NOFOLLOW))
```

`omode & O_NOFOLLOW` evaluates to `0x800` (non-zero → true) → `!(true)` = **false** → loop never entered.

`open()` returns a file descriptor pointing to the **symlink inode itself**. Reading from this fd returns the raw target path string (e.g., `"real_file"`).

> **Use case:** `ls -l` uses `O_NOFOLLOW` to inspect what a symlink points to rather than opening the target.

---

### Variation F — Symlink pointing to a directory

```
mkdir("mydir")
symlink("mydir", "dirlink")
open("dirlink", O_RDONLY)
```

Without `O_NOFOLLOW`:
1. `sys_open()` follows the link → `ip` now points to `mydir`.
2. `ip->type == T_DIR` — the directory check triggers.
3. `open()` returns **−1** (cannot open a directory with `O_RDONLY` without `O_DIRECTORY` on Linux, and xv6 rejects it with the existing directory guard).

> **Exam trap:** The tip in the lab says to insert symlink-following code **before** the directory check. If it is placed after, following a symlink to a directory would hit the directory guard before the link is resolved — wrong behaviour.

---

## Topic 4 — `bread()` / `brelse()` Rules

The buffer cache holds a fixed number of in-memory disk buffers (`NBUF = 30`). `bread()` locks a buffer. `brelse()` releases it. Every `bread()` must be paired with exactly one `brelse()`.

---

### Variation A — Correct: release Master Map before loading Secondary Map

```c
struct buf *bp = bread(dev, master_addr);    // load Master Map
uint addr = ((uint*)bp->data)[index1];       // read secondary addr
brelse(bp);                                  // ← release BEFORE loading next
bp = bread(dev, addr);                       // load Secondary Map
uint data = ((uint*)bp->data)[index2];
brelse(bp);                                  // release Secondary Map
```

**Correct.** Each buffer is released before the next one is loaded.

---

### Variation B — Bug: missing `brelse()` on Master Map

```c
struct buf *bp = bread(dev, master_addr);
uint addr = ((uint*)bp->data)[index1];
// forgot brelse(bp) here
bp = bread(dev, addr);                       // bp now points to Secondary Map
                                             // Master Map buffer is LEAKED
uint data = ((uint*)bp->data)[index2];
brelse(bp);
```

**Bug.** The Master Map buffer is never released. After this code runs ~30 times, the buffer cache is exhausted and the kernel panics with `bget: no buffers`.

---

### Variation C — Bug: `brelse()` called before reading data

```c
struct buf *bp = bread(dev, secondary_addr);
brelse(bp);                                  // ← released too early!
uint data = ((uint*)bp->data)[index2];       // bp->data may be overwritten
```

**Bug.** After `brelse()`, the buffer can be immediately claimed by another `bread()` call in another process. Reading `bp->data` after release accesses memory that may now contain a completely different disk block's data.

---

### Variation D — Bug: same block loaded twice simultaneously

```c
struct buf *bp1 = bread(dev, block_x);
struct buf *bp2 = bread(dev, block_x);       // ← same block number!
```

**Bug — Deadlock.** `bp1` holds the lock on `block_x`. `bread(dev, block_x)` tries to acquire the same lock — it spins forever. The kernel hangs.

---

### Variation E — Correct: two different blocks held simultaneously

```c
struct buf *bp1 = bread(dev, block_a);
struct buf *bp2 = bread(dev, block_b);       // different block — OK
// use both
brelse(bp1);
brelse(bp2);
```

**Correct.** Holding two different blocks simultaneously is allowed as long as the total does not exceed `NBUF = 30`. `brelse` order does not matter.

---

### Variation F — `log_write()` and `brelse()` ordering

```c
// WRONG order:
brelse(bp);
log_write(bp);     // ← bp is no longer yours after brelse!

// CORRECT order:
log_write(bp);     // mark dirty first, while you still own the buffer
brelse(bp);        // THEN release
```

> **Rule:** `log_write(bp)` must always be called **before** `brelse(bp)`. `log_write` marks the buffer as needing to be written to the write-ahead log on the next `commit()`. After `brelse`, you no longer own the buffer — marking it dirty is undefined behaviour.

---

## Topic 5 — Struct Layout Correctness (`dinode` vs `inode`)

`struct dinode` (on-disk in `kernel/fs.h`) and `struct inode` (in-memory in `kernel/file.h`) must have matching `addrs[]` array sizes. `ilock()` copies raw bytes from the on-disk dinode into the in-memory inode without any field-by-field conversion.

---

### Variation A — Both updated correctly (Task 1 complete)

```c
// kernel/fs.h
struct dinode {
  ...
  uint addrs[NDIRECT+2];   // 13 entries: 11 direct + 1 singly + 1 doubly
};

// kernel/file.h
struct inode {
  ...
  uint addrs[NDIRECT+2];   // 13 entries — matches dinode
};
```

**Correct.** `ilock()` copies 13 × 4 = 52 bytes of addresses correctly.

---

### Variation B — `dinode` updated, `inode` not updated (common mistake)

```c
// kernel/fs.h
struct dinode {
  uint addrs[NDIRECT+2];   // 13 entries — correct
};

// kernel/file.h
struct inode {
  uint addrs[NDIRECT+1];   // 12 entries — WRONG, not updated
};
```

**Bug.** `ilock()` copies the on-disk dinode into the in-memory inode. The on-disk struct is 52 bytes of addresses, but the in-memory struct only has room for 48 bytes. The 13th address (`addrs[12]`, the doubly-indirect pointer) **overflows into the next field** in `struct inode`, silently corrupting it. All doubly-indirect lookups read garbage.

---

### Variation C — `inode` updated, `dinode` not updated

```c
// kernel/fs.h
struct dinode {
  uint addrs[NDIRECT+1];   // 12 entries — WRONG, not updated
};

// kernel/file.h
struct inode {
  uint addrs[NDIRECT+2];   // 13 entries — correct
};
```

**Bug.** The on-disk dinode only has 12 address slots. `mkfs` builds the file system with this smaller structure. When a file grows into the doubly-indirect zone, the kernel writes a doubly-indirect pointer into `addrs[12]` in memory — but when this is flushed to disk, the dinode does not have a 13th slot. The doubly-indirect pointer is written into whatever follows the dinode on disk (likely another inode's fields), causing corruption.

---

### Variation D — `NDIRECT` changed but `fs.img` not deleted

After changing `NDIRECT` from 12 to 11, the existing `fs.img` on disk was built with the old 12-direct layout. Running `make qemu` without deleting `fs.img` boots xv6 with a mismatched file system.

**Symptom:** Kernel reads old dinode structures with `NDIRECT=12` layout but the new code expects `NDIRECT=11`. Every inode's address mapping is off by one slot — all file lookups read wrong block numbers. The file system is silently corrupted.

**Fix:**
```
$ rm fs.img
$ make clean && make qemu
```

---

## Topic 6 — `nmeta` Edge Cases

The `nmeta` formula:
```
nmeta = 1 (boot) + 1 (super) + 30 (log) + 13 (inode blocks) + bitmap_blocks
bitmap_blocks = ceil(FSSIZE / 8192)
```

---

### Variation A — FSSIZE = 8192 (exactly one bitmap block's worth)

```
bitmap_blocks = ceil(8192 / 8192) = 1
nmeta         = 1+1+30+13+1 = 46
data_blocks   = 8192 - 46 = 8146
```

```
nmeta 46 (..., bitmap blocks 1) blocks 8146 total 8192
```

---

### Variation B — FSSIZE = 8193 (one block beyond the boundary)

```
bitmap_blocks = ceil(8193 / 8192) = 2
nmeta         = 1+1+30+13+2 = 47
data_blocks   = 8193 - 47 = 8146
```

```
nmeta 47 (..., bitmap blocks 2) blocks 8146 total 8193
```

> **Exam trap:** FSSIZE 8192 and 8193 have different `nmeta` values (46 vs 47) but the same number of data blocks (8146) because the extra bitmap block consumes the one extra FSSIZE block. This is a classic off-by-one question.

---

### Variation C — FSSIZE = 16384

```
bitmap_blocks = ceil(16384 / 8192) = 2
nmeta         = 1+1+30+13+2 = 47
data_blocks   = 16384 - 47 = 16337
```

```
nmeta 47 (..., bitmap blocks 2) blocks 16337 total 16384
```

---

### Variation D — FSSIZE = 200000 (Task 1 value)

```
bitmap_blocks = ceil(200000 / 8192) = 25
nmeta         = 1+1+30+13+25 = 70
data_blocks   = 200000 - 70 = 199930
```

```
nmeta 70 (..., bitmap blocks 25) blocks 199930 total 200000
```

---

### Variation E — FSSIZE = 2000 (original, before Task 1)

```
bitmap_blocks = ceil(2000 / 8192) = 1
nmeta         = 1+1+30+13+1 = 46
data_blocks   = 2000 - 46 = 1954
```

```
nmeta 46 (..., bitmap blocks 1) blocks 1954 total 2000
```

> **Exam trap:** FSSIZE 2000 and FSSIZE 8192 both produce `nmeta = 46` (both need only 1 bitmap block). But their data block counts are very different: 1954 vs 8146. `nmeta` alone does not tell you the disk size.

---

### Variation F — FSSIZE = 24576 (3 × 8192)

```
bitmap_blocks = ceil(24576 / 8192) = 3
nmeta         = 1+1+30+13+3 = 48
data_blocks   = 24576 - 48 = 24528
```

```
nmeta 48 (..., bitmap blocks 3) blocks 24528 total 24576
```

---

## Topic 7 — `O_NOFOLLOW` Flag Bitwise Analysis

`O_NOFOLLOW` must occupy a bit position not used by any existing flag. Flags are combined with bitwise OR, so any overlap causes silent misidentification.

**Existing flags:**
```
O_RDONLY = 0x000  (0000 0000 0000)
O_WRONLY = 0x001  (0000 0000 0001)
O_RDWR   = 0x002  (0000 0000 0010)
O_CREATE = 0x200  (0010 0000 0000)
O_TRUNC  = 0x400  (0100 0000 0000)
```

---

### Variation A — Correct value: `0x800`

```
O_NOFOLLOW = 0x800  (1000 0000 0000)
```

No bit overlap with any existing flag. `flags & O_NOFOLLOW` returns `0x800` if and only if `O_NOFOLLOW` was explicitly passed — clean isolation.

---

### Variation B — Wrong value: `0x001` (same as `O_WRONLY`)

```
O_NOFOLLOW = 0x001
```

Any `open()` call with `O_WRONLY` automatically sets the `O_NOFOLLOW` bit. Result: **all write-mode opens never follow symlinks**. Files written via a symlink silently open the symlink itself, not the target. No compile error — pure runtime breakage.

---

### Variation C — Wrong value: `0x003` (overlaps `O_WRONLY` and `O_RDWR`)

```
O_NOFOLLOW = 0x003
O_RDWR     = 0x002
```

`0x003 & 0x002 = 0x002` — any read-write open partially matches `O_NOFOLLOW`. The check `omode & O_NOFOLLOW` returns non-zero for all `O_RDWR` opens, causing symlinks to never be followed on read-write opens. **Silent partial breakage.**

---

### Variation D — Wrong value: `0x600` (spans `O_CREATE` and `O_TRUNC`)

```
O_NOFOLLOW = 0x600
O_CREATE   = 0x200
O_TRUNC    = 0x400
O_CREATE | O_TRUNC = 0x600 = O_NOFOLLOW (accidentally)
```

Any `open(..., O_CREATE | O_TRUNC)` call sets the full `O_NOFOLLOW` bit pattern. Creating and truncating a file via a symlink silently opens the symlink itself. **Silent breakage on all file creation operations.**

---

### Variation E — Test: what does `flags & O_NOFOLLOW` return for various flag combinations?

| `omode` passed | `omode & 0x800` | No-follow triggered? |
|---|---|---|
| `O_RDONLY` (0x000) | 0x000 | No |
| `O_WRONLY` (0x001) | 0x000 | No |
| `O_RDWR` (0x002) | 0x000 | No |
| `O_RDONLY \| O_NOFOLLOW` (0x800) | 0x800 | **Yes** |
| `O_WRONLY \| O_NOFOLLOW` (0x801) | 0x800 | **Yes** |
| `O_CREATE \| O_NOFOLLOW` (0xA00) | 0x800 | **Yes** |
| `O_CREATE \| O_TRUNC` (0x600) | 0x000 | No |

> `O_CREATE | O_TRUNC` does **not** trigger `O_NOFOLLOW` when `O_NOFOLLOW = 0x800` — confirming 0x800 is the correct value.

---

## Quick Reference Cheat Sheet

| Concept | Key value / formula |
|---|---|
| NDIRECT (after Task 1) | 11 |
| NINDIRECT | 256 (= 1024 / 4) |
| NDINDIRECT | 256 × 256 = 65,536 |
| MAXFILE | 11 + 256 + 65,536 = **65,803** |
| Direct range | blocks 0 – 10 |
| Singly-indirect range | blocks 11 – 266 |
| Doubly-indirect range | blocks 267 – 65,802 |
| `index1` formula | `(bn - 267) / 256` |
| `index2` formula | `(bn - 267) % 256` |
| Full file bfree() calls | **66,061** |
| Bitmap block coverage | 8,192 blocks per bitmap block |
| nmeta (FSSIZE=2000) | 46 |
| nmeta (FSSIZE=200000) | 70 |
| Max symlink depth | 10 |
| `O_NOFOLLOW` value | `0x800` |
| `T_SYMLINK` value | 4 |
| `addrs[]` size (after) | `NDIRECT + 2` = **13** |
| Buffer cache size (NBUF) | 30 |
| `log_write` before `brelse` | Always — mark dirty before releasing |
| dinode and inode addrs[] | Must always match |
