---
layout: page
title: "Lab Predictions - lab3-t5r9"
lab: lab3
description: "Exam predictions for Lab3-w8: nmeta calculation, O_NOFOLLOW flags, bmap index tracing, itrunc block counting, and circular link deadlock."
---

# Lab3-w8 Exam Predictions

---

## Topic 1 — `nmeta` Calculation (FSSIZE Variants)

The `nmeta` line printed by `mkfs` counts all metadata blocks: boot + super + log blocks + inode blocks + bitmap blocks.

**Formula recap (xv6 defaults):**
- Boot: 1
- Super: 1
- Log: 30
- Inode blocks: `ceil(NINODES * sizeof(dinode) / BSIZE)` → typically **13**
- Bitmap blocks: `ceil(FSSIZE / (BSIZE * 8))`
- Data blocks: `FSSIZE - nmeta`

---

### Variation A — FSSIZE = 2000 (original, before Task 1)

| Parameter | Value |
|---|---|
| FSSIZE | 2000 |
| Bitmap blocks | `ceil(2000 / 8192)` = **1** |
| nmeta | 1+1+30+13+1 = **46** |
| Data blocks | 2000 − 46 = **1954** |

**Expected output:**
```
nmeta 46 (boot, super, log blocks 30 inode blocks 13, bitmap blocks 1) blocks 1954 total 2000
```

---

### Variation B — FSSIZE = 200000 (after Task 1 change in `param.h`)

| Parameter | Value |
|---|---|
| FSSIZE | 200000 |
| Bitmap blocks | `ceil(200000 / 8192)` = **25** |
| nmeta | 1+1+30+13+25 = **70** |
| Data blocks | 200000 − 70 = **199930** |

**Expected output:**
```
nmeta 70 (boot, super, log blocks 30 inode blocks 13, bitmap blocks 25) blocks 199930 total 200000
```

> **Exam trap:** Students often forget bitmap blocks scale with FSSIZE. Each bitmap block tracks 8192 blocks (1024 bytes × 8 bits). Going from 2000 → 200000 pushes bitmap blocks from 1 → 25.

---

### Variation C — FSSIZE = 100000

| Parameter | Value |
|---|---|
| FSSIZE | 100000 |
| Bitmap blocks | `ceil(100000 / 8192)` = **13** |
| nmeta | 1+1+30+13+13 = **58** |
| Data blocks | 100000 − 58 = **99942** |

---

### Variation D — FSSIZE = 50000

| Parameter | Value |
|---|---|
| FSSIZE | 50000 |
| Bitmap blocks | `ceil(50000 / 8192)` = **7** |
| nmeta | 1+1+30+13+7 = **52** |
| Data blocks | 50000 − 52 = **49948** |

---

## Topic 2 — `O_NOFOLLOW` Flag (Bitwise Correctness)

`O_NOFOLLOW` must not overlap with any existing `open()` flags defined in `kernel/fcntl.h`.

### Existing flags (standard xv6):

| Flag | Value (hex) | Value (binary) |
|---|---|---|
| `O_RDONLY` | `0x000` | `0000 0000` |
| `O_WRONLY` | `0x001` | `0000 0001` |
| `O_RDWR` | `0x002` | `0000 0010` |
| `O_CREATE` | `0x200` | `0010 0000 0000` |
| `O_TRUNC` | `0x400` | `0100 0000 0000` |

### Correct definition:

```c
#define O_NOFOLLOW 0x800
```

> **Why 0x800?** It is the next unused power-of-2 bit: `1000 0000 0000`. No bitwise overlap with any existing flag, so `flags & O_NOFOLLOW` isolates it cleanly.

### Variation — What if `O_NOFOLLOW` was set to `0x002`?

`0x002` is `O_RDWR`. Bitwise OR of `O_RDWR | O_NOFOLLOW` would be indistinguishable from just `O_RDWR`. **The kernel could not tell them apart** — any read-write open would accidentally trigger the no-follow path. This is why non-overlapping bit assignment is mandatory.

### Variation — What if `O_NOFOLLOW` was set to `0x600`?

`0x600 = 0x200 | 0x400 = O_CREATE | O_TRUNC`. Overlaps two existing flags. A file opened with `O_CREATE | O_TRUNC` would accidentally set `O_NOFOLLOW` bits, causing symlinks to never be followed when creating/truncating files. **Silent breakage** — no compile error, just wrong runtime behaviour.

---

## Topic 3 — `bmap()` Index Tracing (Doubly-Indirect)

After Task 1, the address layout is:

| Slot | Range | Type |
|---|---|---|
| `addrs[0..10]` | Blocks 0–10 | Direct (11 blocks) |
| `addrs[11]` | Blocks 11–266 | Singly-indirect (256 blocks) |
| `addrs[12]` | Blocks 267–65802 | Doubly-indirect (256×256 = 65536 blocks) |

For doubly-indirect, the logical block number `bn` is **relative to the start of the doubly-indirect zone** (i.e., subtract 11 + 256 = 267 first).

**Formulas:**
```
bn        = logical_block - 267
index1    = bn / 256        ← which Secondary Map Book
index2    = bn % 256        ← which slot inside that Secondary Map Book
```

---

### Variation A — Logical block 500

```
bn     = 500 − 267 = 233
index1 = 233 / 256 = 0      ← Secondary Map Book #0
index2 = 233 % 256 = 233    ← slot 233 inside Secondary Map Book #0
```

Path: `addrs[12]` → Master Map[0] → Secondary Map #0[233] → Data Block

---

### Variation B — Logical block 1000

```
bn     = 1000 − 267 = 733
index1 = 733 / 256 = 2      ← Secondary Map Book #2
index2 = 733 % 256 = 221    ← slot 221 inside Secondary Map Book #2
```

Path: `addrs[12]` → Master Map[2] → Secondary Map #2[221] → Data Block

---

### Variation C — Logical block 267 (first doubly-indirect block)

```
bn     = 267 − 267 = 0
index1 = 0 / 256 = 0
index2 = 0 % 256 = 0
```

Path: `addrs[12]` → Master Map[0] → Secondary Map #0[0] → Data Block

> **Exam trap:** Students sometimes forget to subtract 267 before computing indices, leading to wildly wrong answers. Always adjust `bn` relative to the start of the doubly-indirect region.

---

### Variation D — Logical block 65802 (last possible block)

```
bn     = 65802 − 267 = 65535
index1 = 65535 / 256 = 255   ← Secondary Map Book #255
index2 = 65535 % 256 = 255   ← slot 255 inside Secondary Map Book #255
```

Path: `addrs[12]` → Master Map[255] → Secondary Map #255[255] → Data Block

---

### Variation E — Logical block 266 (last singly-indirect block, NOT doubly)

```
266 < 267  →  falls in singly-indirect zone, NOT doubly-indirect
bn_single = 266 − 11 = 255
```

Path: `addrs[11]` → Map Block[255] → Data Block

> **Exam trap:** Block 266 looks close to the doubly-indirect zone but is still the last slot of the singly-indirect map. Off-by-one errors here corrupt the file system.

---

## Topic 4 — `itrunc()` Block Counting

`itrunc()` must free every allocated block when a file is deleted. For a **fully allocated** file (all 65,803 blocks used), the total `bfree()` calls are:

| Zone | Blocks freed | `bfree()` calls |
|---|---|---|
| Direct | 11 data blocks | 11 |
| Singly-indirect | 1 map block + 256 data blocks | 257 |
| Doubly-indirect data | 256 × 256 data blocks | 65536 |
| Secondary Map Books | 256 secondary map blocks | 256 |
| Master Map Book | 1 master map block | 1 |
| **Total** | | **66061** |

---

### Variation A — File uses only direct blocks (e.g., 5 blocks)

`bfree()` calls: **5** (just the direct blocks; no map blocks allocated)

---

### Variation B — File uses direct + full singly-indirect (11 + 256 = 267 blocks)

`bfree()` calls:
- 11 direct data blocks
- 256 singly-indirect data blocks
- 1 singly-indirect map block

**Total: 268**

---

### Variation C — File uses direct + singly-indirect + partial doubly-indirect

Suppose the doubly-indirect zone has **2 Secondary Map Books** allocated, each **fully used** (256 data blocks each):

`bfree()` calls:
- 11 (direct)
- 257 (singly-indirect: 1 map + 256 data)
- 2 × 256 = 512 (data blocks across 2 secondary maps)
- 2 (the 2 secondary map blocks themselves)
- 1 (master map block)

**Total: 783**

---

### Variation D — Common `itrunc()` bug: forgetting to free the map blocks themselves

A student frees all data blocks but forgets to call `bfree()` on the Secondary Map Books and the Master Map Book. For a full file, this leaks:
- **256** secondary map blocks
- **1** master map block

**257 blocks permanently leaked** from the free pool — disk fills up silently over time.

---

## Topic 5 — Circular Symlink & Deadlock Scenarios

### Scenario A — Simple cycle (A → B → A)

```
symlink("b", "a")   // "a" points to "b"
symlink("a", "b")   // "b" points to "a"
open("a", O_RDONLY)
```

**Without depth guard:** kernel loops forever — `open("a")` → read link → `open("b")` → read link → `open("a")` → ...

**With depth guard (max 10):** after 10 iterations, `open()` returns **−1** (error). `symlinktest` verifies this is the correct behaviour.

---

### Scenario B — Long chain (link4 → link3 → link2 → link1 → real file)

Depth traversed: **4 jumps**. Well within the limit of 10. `open()` succeeds and returns the file descriptor of the real file.

---

### Scenario C — Chain of exactly 10 (boundary)

A chain of 10 symlinks all pointing forward, with the 10th pointing to a real file:

```
link10 → link9 → link8 → ... → link1 → real_file
```

Depth counter hits 10 on `link1`. Whether this succeeds depends on implementation:
- If the guard triggers **before** reading the final real file (i.e., `depth >= 10` → error), this **fails**.
- If the guard triggers **after** confirming the target is not a symlink, this **succeeds**.

> **Exam question:** What should the guard condition be — `depth > 10` or `depth >= 10`? The standard safe choice is to **error at depth 10** (i.e., allow at most 9 redirections) to match POSIX `ELOOP` behaviour.

---

### Scenario D — Deadlock via improper locking

```c
// Buggy code inside sys_open() while following a symlink:
ilock(ip);                     // lock current symlink inode
// ... read path from ip ...
struct inode *next = namei(path);
ilock(next);                   // ← DEADLOCK if next == ip (self-referential link)
iunlock(ip);                   // never reached
```

**Fix:** Always `iunlock(ip)` and `iput(ip)` **before** calling `namei()` and locking the next inode. The corrected sequence:

```c
ilock(ip);
readi(ip, ...);        // read the symlink target path
iunlock(ip);           // release lock BEFORE following the path
iput(ip);              // drop reference on old inode
ip = namei(new_path);  // find next inode
// loop back to ilock(ip) at top of while loop
```

---

### Scenario E — Dangling symlink (target deleted)

```
create "real_file"
symlink("real_file", "mylink")
unlink("real_file")
open("mylink", O_RDONLY)   // what happens?
```

`namei("real_file")` returns **0** (inode not found). `sys_open()` must return **−1**. This is tested indirectly by `symlinktest` — a symlink pointing to a non-existent file must not panic the kernel, it must return an error gracefully.

---

## Quick Reference Cheat Sheet

| Concept | Key value / formula |
|---|---|
| NDIRECT (after) | 11 |
| NINDIRECT | 256 (= BSIZE / sizeof(uint) = 1024/4) |
| NDINDIRECT | 256 × 256 = 65536 |
| MAXFILE (after) | 11 + 256 + 65536 = **65803** |
| Doubly-indirect start block | 11 + 256 = **267** |
| bmap index1 formula | `(bn - 267) / 256` |
| bmap index2 formula | `(bn - 267) % 256` |
| Bitmap block coverage | 8192 blocks per bitmap block |
| Max symlink depth | 10 |
| O_NOFOLLOW value | `0x800` |
| `addrs[]` size (after) | `NDIRECT + 2` = **13** |
