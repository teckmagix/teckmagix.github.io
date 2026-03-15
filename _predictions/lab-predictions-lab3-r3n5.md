---
layout: page
title: "Lab Predictions - lab3-r3n5"
lab: lab3
description: "Exam predictions for Lab3-w8: balloc/bfree disk lifecycle, bmap full path tracing, sys_symlink argument handling, namei path resolution, iget/iput reference counting, begin_op/end_op transactions, and addrs[] slot indexing after NDIRECT change."
---

# Lab3-w8 Exam Predictions (Fresh Topics)

---

## Topic 1 — `balloc()` / `bfree()` Disk Block Lifecycle

`balloc(dev)` allocates a free disk block and returns its block number. `bfree(dev, blockno)` marks a block as free. These two functions are the only way `bmap()` and `itrunc()` interact with the free block pool.

**Key rules:**
- `balloc()` returns a **zeroed** block — all bytes are 0.
- A zero entry in an indirect block means "not yet allocated".
- `bfree()` does not zero the block — the old data remains on disk until the block is reused.
- `balloc()` returns **0** if the disk is full — callers must check this.

---

### Variation A — Normal allocation and free cycle

```
balloc(dev)  → returns block 105  (zeroed)
... file writes data to block 105 ...
bfree(dev, 105)  → block 105 returned to free pool
balloc(dev)  → may return block 105 again (or a different free block)
```

The block is not zeroed by `bfree()`. The next `balloc()` call that returns block 105 gives back a zeroed version (because `balloc` always zeros before returning).

---

### Variation B — `balloc()` returns 0 (disk full)

```c
uint addr = balloc(ip->dev);
if(addr == 0)
    return 0;   // ← must check — do not use addr=0 as a real block address
```

Block 0 is the boot block and is **never** a valid data block. If `balloc()` returns 0, it means no free blocks exist. `bmap()` must return 0 to signal failure. If the caller ignores this and writes to block 0, the boot block is corrupted.

> **Exam trap:** Students forget to check `balloc()` return value. The code compiles fine and appears to work until the disk fills up — then it silently writes to block 0 and corrupts the file system.

---

### Variation C — `bfree()` without a matching `brelse()`

```c
struct buf *bp = bread(dev, addr);
bfree(dev, addr);    // ← freed the disk block
brelse(bp);          // ← now releasing the in-memory buffer for that block
```

**Order matters.** Freeing the disk block before releasing the buffer is technically wrong — the buffer still holds a reference to a now-free block. Another `balloc()` call could reuse the block before `brelse()` releases it, causing two processes to simultaneously hold the same block. The correct order is always `brelse()` first, then `bfree()`.

---

### Variation D — What happens to zeros in indirect blocks after `balloc()`?

`balloc()` zeros the new block before returning it. For an indirect block (a map block), this means all 256 entries start as 0. When `bmap()` later reads a map block and checks `a[index] == 0`, a zero means "this data block has not been allocated yet" — and it correctly calls `balloc()` to create it on demand.

This is why zeroing is critical: without it, old random data in a recycled map block could look like valid block addresses, causing `bmap()` to access completely wrong disk blocks.

---

### Variation E — `balloc()` call count for writing a file of 300 blocks from scratch

Starting from an empty file:

| Zone | Blocks allocated | `balloc()` calls |
|---|---|---|
| Direct (11) | 11 data blocks | 11 |
| Singly-indirect map | 1 map block | 1 |
| Singly-indirect data | 256 data blocks | 256 |
| Doubly-indirect Master Map | 1 master map block | 1 |
| Doubly-indirect Secondary Map #0 | 1 secondary map block | 1 |
| Doubly-indirect data (300 - 267 = 33 blocks) | 33 data blocks | 33 |
| **Total** | | **303** |

> **Why 303 and not 300?** Three extra `balloc()` calls are for the map blocks themselves: singly-indirect map (1), Master Map (1), Secondary Map #0 (1). Map blocks are real disk blocks that must also be allocated.

---

## Topic 2 — Full `bmap()` Path Tracing (Step by Step)

For a given logical block number, trace every step `bmap()` takes: which tier is entered, which slots are read, which `bread()`/`brelse()`/`balloc()` calls are made, and what is returned.

---

### Variation A — Block 5 (direct, already allocated)

Assume `ip->addrs[5] = 72` (block 72 already allocated).

```
bn = 5
5 < NDIRECT (11) → direct zone

return ip->addrs[5]  →  return 72
```

No `bread()`, no `brelse()`, no `balloc()`. One array lookup.

---

### Variation B — Block 15 (singly-indirect, map exists, data block exists)

Assume `ip->addrs[11] = 88` (singly-indirect map at block 88).
Assume `map[4] = 120` (data block at block 120).

```
bn = 15
15 >= NDIRECT (11) → singly-indirect zone
bn -= NDIRECT  →  bn = 4

ip->addrs[11] = 88 ≠ 0  →  no balloc needed
bp = bread(dev, 88)
a = (uint*)bp->data
a[4] = 120 ≠ 0  →  no balloc needed
brelse(bp)

return 120
```

One `bread()`, one `brelse()`, zero `balloc()` calls.

---

### Variation C — Block 15 (singly-indirect, data block NOT yet allocated)

Assume `ip->addrs[11] = 88` (map exists), but `map[4] = 0` (no data block yet).

```
bn = 4 (after subtracting NDIRECT)

bp = bread(dev, 88)
a[4] = 0  →  balloc(dev) → returns 201
a[4] = 201
log_write(bp)    ← mark map dirty
brelse(bp)

return 201
```

One `bread()`, one `balloc()`, one `log_write()`, one `brelse()`.

---

### Variation D — Block 300 (doubly-indirect, nothing allocated yet)

Starting from a completely empty file — `addrs[12] = 0`.

```
bn = 300
300 >= NDIRECT + NINDIRECT (267) → doubly-indirect zone
bn -= (NDIRECT + NINDIRECT)  →  bn = 33
index1 = 33 / 256 = 0
index2 = 33 % 256 = 33

Step 1: Master Map
  addrs[12] = 0  →  balloc() → master_addr = 300
  addrs[12] = 300
  bp = bread(dev, 300)
  a[0] = 0  →  balloc() → sec_addr = 301
  a[0] = 301
  log_write(bp)     ← Master Map updated
  brelse(bp)

Step 2: Secondary Map
  bp = bread(dev, 301)
  a[33] = 0  →  balloc() → data_addr = 302
  a[33] = 302
  log_write(bp)     ← Secondary Map updated
  brelse(bp)

return 302
```

Three `balloc()` calls (Master Map, Secondary Map #0, data block), two `bread()` + `brelse()` pairs, two `log_write()` calls.

---

### Variation E — Block 300 (doubly-indirect, Master Map exists, Secondary Map exists, data block missing)

Assume `addrs[12] = 300`, `master[0] = 301`, `secondary[33] = 0`.

```
bp = bread(dev, 300)   ← load Master Map
a[0] = 301 ≠ 0         ← Secondary Map exists, no balloc
brelse(bp)             ← release Master Map

bp = bread(dev, 301)   ← load Secondary Map
a[33] = 0              ← data block missing
balloc() → 500
a[33] = 500
log_write(bp)
brelse(bp)

return 500
```

Zero `balloc()` for map blocks (already exist), one `balloc()` for data block, two `bread()`/`brelse()` pairs.

---

## Topic 3 — `sys_symlink()` Argument Handling

`sys_symlink(const char *target, const char *path)` receives two strings from user space. The kernel must safely fetch them using `argstr()` before doing anything with them.

---

### Variation A — Argument order: which is target, which is path?

```c
uint64
sys_symlink(void)
{
  char target[MAXPATH], path[MAXPATH];

  if(argstr(0, target, MAXPATH) < 0)   // argument 0 = target
    return -1;
  if(argstr(1, path, MAXPATH) < 0)     // argument 1 = path (new symlink name)
    return -1;
  ...
}
```

In C: `symlink(target, path)` — the **target** is what the link points TO, the **path** is the name of the new symlink file being created.

Example: `symlink("real_file", "mylink")` → `mylink` is a new file that points to `real_file`.

> **Exam trap:** Swapping the two `argstr()` calls writes the symlink name as the stored path and the target path as the file name. The symlink file is created with the wrong name and stores the wrong target — both silently wrong.

---

### Variation B — What does `argstr(0, target, MAXPATH)` actually do?

`argstr(n, buf, max)` reads the n-th argument from the trap frame, interprets it as a user-space pointer to a string, and **copies** the string from user memory into the kernel buffer `buf` (up to `max` bytes).

Steps:
1. Reads register `a0` from the trap frame (for argument 0).
2. Validates the user pointer — returns −1 if invalid.
3. Copies bytes from user address space into `target[]` in kernel space.

Without this copy, the kernel would dereference a raw user pointer — which is unsafe and could read from any address the user controls.

---

### Variation C — What if `argstr()` returns −1?

```c
if(argstr(0, target, MAXPATH) < 0)
    return -1;
```

`argstr()` returns −1 if:
- The user pointer is null.
- The string length exceeds `MAXPATH`.
- The user pointer points to unmapped memory.

Returning −1 immediately is correct — no `begin_op()` has been called yet, so no transaction needs to be rolled back.

> **Exam trap:** If `begin_op()` is called before `argstr()` and `argstr()` fails, you must call `end_op()` before returning −1. The safer pattern (used in the reference implementation) is to call `argstr()` **before** `begin_op()` so no transaction is open on failure.

---

### Variation D — `writei()` to store the target path in the symlink inode

```c
if(writei(ip, 0, (uint64)target, 0, strlen(target)) != strlen(target)){
    iunlockput(ip);
    end_op();
    return -1;
}
```

`writei(ip, user_src, src, off, n)`:
- `ip` — inode to write into
- `user_src = 0` — source is a **kernel** address (not user space)
- `src = (uint64)target` — the kernel buffer holding the path string
- `off = 0` — write starting at byte 0 of the inode's data
- `n = strlen(target)` — number of bytes to write

The target path (e.g., `"/usr/bin/python3"`) is stored as raw bytes in the inode's data blocks — no null terminator is guaranteed unless `n` includes it.

> **Exam trap:** Using `user_src = 1` (user source) when the data is in a kernel buffer causes `writei()` to try to copy from user address space — it reads garbage or panics.

---

### Variation E — `readi()` to read the path back in `sys_open()`

```c
char path_buf[MAXPATH];
int n = readi(ip, 0, (uint64)path_buf, 0, MAXPATH);
path_buf[n] = '\0';   // ensure null termination
```

`readi(ip, user_dst, dst, off, n)`:
- `user_dst = 0` — destination is a **kernel** buffer
- `dst = (uint64)path_buf` — where to copy into
- `off = 0` — read from byte 0 of inode data
- `n = MAXPATH` — read up to MAXPATH bytes

The returned value `n` is the number of bytes actually read. The null terminator must be manually added because `writei()` does not store it.

> **Exam trap:** Forgetting to null-terminate `path_buf` after `readi()` causes `namei(path_buf)` to scan beyond the stored string into uninitialized memory — producing an incorrect path and returning 0 (inode not found) even when the target exists.

---

## Topic 4 — `namei()` Path Resolution

`namei(path)` converts a path string (e.g., `"/usr/bin/sh"`) into the inode for the file at that path. It returns a **locked** inode on success, or `0` if the file does not exist.

---

### Variation A — Absolute vs relative paths

```c
namei("/real_file")    // absolute — starts from root inode
namei("real_file")     // relative — starts from current working directory
```

Both are valid in xv6 symlink target strings. `namei()` handles both — it checks if the first character is `/` to decide the starting inode.

> If a symlink stores a relative path, resolution depends on the current working directory of the process that calls `open()`. A symlink storing an absolute path always resolves the same way regardless of cwd.

---

### Variation B — `namei()` returns 0 (file not found)

```c
ip = namei("nonexistent_file");
if(ip == 0){
    end_op();
    return -1;
}
```

`namei()` returns 0 in two cases:
1. The file does not exist.
2. A directory component in the path does not exist (e.g., `namei("/a/b/c")` when `/a/b/` does not exist).

Both cases must be handled identically — return −1 without panicking.

---

### Variation C — `namei()` vs `nameiparent()`

| Function | Returns | Used for |
|---|---|---|
| `namei(path)` | Inode of the file at `path` | Opening a file, following a symlink |
| `nameiparent(path, name)` | Inode of the **parent directory**, copies final component into `name` | Creating a file (`create()`), unlinking |

`sys_symlink()` calls `create(path, T_SYMLINK, ...)` which internally uses `nameiparent()` to find the directory where the new symlink should be created, then creates an entry in that directory.

---

### Variation D — What `namei()` returns for each file type

| File type | `namei()` return | `ip->type` after `ilock()` |
|---|---|---|
| Regular file | inode pointer | `T_FILE` (= 2) |
| Directory | inode pointer | `T_DIR` (= 1) |
| Symlink | inode pointer | `T_SYMLINK` (= 4) |
| Device | inode pointer | `T_DEVICE` (= 3) |
| Does not exist | `0` | N/A |

`namei()` returns the inode **without following symlinks** — it is `sys_open()` that decides whether to follow based on the type and flags.

---

### Variation E — Why `ilock()` must be called after `namei()`

`namei()` returns an inode pointer but does **not** lock it. The inode's fields (`type`, `size`, `addrs[]`) must not be read until `ilock()` is called, because another process might be modifying them concurrently.

```c
ip = namei(path);
if(ip == 0) return -1;
ilock(ip);              // ← must lock before reading ip->type
if(ip->type == T_SYMLINK) ...
```

Reading `ip->type` without `ilock()` is a data race — the type might be partially written by a concurrent `create()` call.

---

## Topic 5 — `iget()` / `iput()` Reference Counting

Every inode has a reference count (`ip->ref`) tracking how many kernel structures currently hold a pointer to it. `iget()` increments the count (find or allocate an in-memory inode). `iput()` decrements it and frees the inode if count reaches 0.

---

### Variation A — Normal open/close cycle

```
iget()   → ref = 1   (sys_open finds the inode)
iget()   → ref = 2   (another process opens the same file)
iput()   → ref = 1   (first process closes)
iput()   → ref = 0   (second process closes → inode freed from memory)
```

The inode is only freed from the in-memory inode table when `ref` reaches 0.

---

### Variation B — `iput()` forgotten in symlink traversal

```c
// Inside sys_open() symlink loop:
iunlock(ip);
// forgot iput(ip) here
ip = namei(new_path);
ilock(ip);
```

Without `iput(ip)`, the previous inode's reference count is never decremented. After following many symlinks across many `open()` calls, the inode table fills up. New `iget()` calls panic with `iget: no inodes`.

> **Fix:** Always call `iunlock(ip)` followed immediately by `iput(ip)` before moving to the next inode. Or use the combined `iunlockput(ip)` which does both.

---

### Variation C — `iunlockput()` vs `iunlock()` + `iput()` separately

```c
// These are equivalent:
iunlockput(ip);

// vs:
iunlock(ip);
iput(ip);
```

`iunlockput()` is a convenience wrapper. It is used when you are done with an inode entirely (both releasing the lock and dropping the reference). In contrast, if you need to read more data after unlocking (e.g., calling `namei()` before locking the next inode), use the two-step version to be explicit.

---

### Variation D — Reference count during symlink chain traversal

For `open("c")` where `c → b → a → real_file`:

| Step | Action | ref changes |
|---|---|---|
| 1 | `namei("c")` → `iget()` | c: ref=1 |
| 2 | `ilock(c)` | — |
| 3 | Follow link → `iunlockput(c)` | c: ref=0 |
| 4 | `namei("b")` → `iget()` | b: ref=1 |
| 5 | `ilock(b)` | — |
| 6 | Follow link → `iunlockput(b)` | b: ref=0 |
| 7 | `namei("a")` → `iget()` | a: ref=1 |
| 8 | `ilock(a)` | — |
| 9 | Follow link → `iunlockput(a)` | a: ref=0 |
| 10 | `namei("real_file")` → `iget()` | real_file: ref=1 |
| 11 | `ilock(real_file)`, type=T_FILE, loop exits | — |

At end: only `real_file` has ref=1. All intermediate symlink inodes have ref=0.

---

### Variation E — What happens when `iput()` is called with ref=1 and nlink=0?

When `ref` drops to 0 and `ip->nlink == 0` (no directory entries point to this inode), `iput()` calls `itrunc()` to free all data blocks, then marks the inode as free on disk. This is how `unlink()` + `close()` actually frees a file — `unlink()` decrements `nlink` to 0 and `close()` (via `iput()`) triggers the cleanup.

---

## Topic 6 — `begin_op()` / `end_op()` Transaction Rules

xv6 uses a write-ahead log to protect file system consistency. Every sequence of disk writes that must be atomic must be wrapped in `begin_op()` ... `end_op()`.

---

### Variation A — Correct transaction structure in `sys_symlink()`

```c
begin_op();

ip = create(path, T_SYMLINK, 0, 0);
if(ip == 0){
    end_op();        // ← must end before returning on any error
    return -1;
}

writei(ip, 0, (uint64)target, 0, strlen(target));
iunlockput(ip);

end_op();            // ← commit the transaction
return 0;
```

`begin_op()` and `end_op()` must be perfectly balanced — every code path that calls `begin_op()` must eventually call `end_op()`, including all error paths.

---

### Variation B — Missing `end_op()` on error path

```c
begin_op();
ip = create(path, T_SYMLINK, 0, 0);
if(ip == 0)
    return -1;       // ← WRONG: begin_op without matching end_op
```

`begin_op()` increments a counter of active transactions. Without `end_op()`, the counter is never decremented. The log system eventually stalls waiting for the phantom transaction to complete — the kernel hangs.

---

### Variation C — `log_write()` vs direct `bwrite()`

Inside a transaction, disk blocks must be written using `log_write(bp)`, not `bwrite(bp)` directly.

| `bwrite(bp)` | `log_write(bp)` |
|---|---|
| Writes immediately to disk | Adds block to the write-ahead log |
| Not crash-safe | Crash-safe — replayed on reboot if needed |
| No transaction tracking | Tracked by `begin_op`/`end_op` |

Using `bwrite()` inside `bmap()` or `itrunc()` bypasses the log — a crash midway through a multi-block update leaves the file system in a partially updated, inconsistent state.

---

### Variation D — Why `create()` is called inside a transaction

`create(path, T_SYMLINK, 0, 0)` internally:
1. Calls `nameiparent()` to find the parent directory.
2. Calls `ialloc()` to allocate a new inode.
3. Writes a new directory entry linking `path` → new inode.

These are three separate disk writes. They must all succeed or all be rolled back. Wrapping in `begin_op()`/`end_op()` ensures the log records all three writes atomically — if a crash happens after step 1 but before step 3, the log replay on reboot either completes all three or none.

---

### Variation E — `begin_op()` called after `argstr()`

The correct ordering:
```c
// 1. Fetch arguments (no disk I/O, no transaction needed)
argstr(0, target, MAXPATH);
argstr(1, path, MAXPATH);

// 2. Open transaction (only after arguments are safely in kernel buffers)
begin_op();

// 3. File system operations
ip = create(path, T_SYMLINK, 0, 0);
...

// 4. Close transaction
end_op();
```

Calling `begin_op()` before `argstr()` wastes a transaction slot on argument fetching. More importantly, if `argstr()` fails after `begin_op()`, you must remember to call `end_op()` — easy to forget. Fetching arguments before opening the transaction is simpler and cleaner.

---

## Topic 7 — `addrs[]` Slot Indexing After NDIRECT Change

After Task 1, `NDIRECT` changes from 12 to 11. This shifts the positions of the indirect slots in `addrs[]`.

---

### Variation A — Before Task 1: NDIRECT = 12

| `addrs[]` index | Slot type | Points to |
|---|---|---|
| 0 – 11 | 12 direct slots | Data blocks 0–11 |
| 12 | Singly-indirect | Map block → data blocks 12–267 |
| No doubly-indirect | — | — |

---

### Variation B — After Task 1: NDIRECT = 11

| `addrs[]` index | Slot type | Points to |
|---|---|---|
| 0 – 10 | 11 direct slots | Data blocks 0–10 |
| 11 | Singly-indirect | Map block → data blocks 11–266 |
| 12 | Doubly-indirect | Master Map → 256 Secondary Maps → data blocks 267–65802 |

> **Critical:** The singly-indirect slot moves from index 12 to index 11. Any code that hardcodes `addrs[12]` for singly-indirect (instead of `addrs[NDIRECT]`) breaks silently after the change.

---

### Variation C — What happens in `bmap()` if you use `addrs[NDIRECT]` vs `addrs[12]` hardcoded

Before Task 1 (`NDIRECT = 12`): `addrs[NDIRECT]` == `addrs[12]` — correct singly-indirect slot.

After Task 1 (`NDIRECT = 11`): `addrs[NDIRECT]` == `addrs[11]` — still correct. But `addrs[12]` hardcoded now accidentally reads/writes the **doubly-indirect slot** as if it were the singly-indirect slot — corrupting both.

**Lesson:** Always use `addrs[NDIRECT]` for singly-indirect and `addrs[NDIRECT+1]` for doubly-indirect. Never hardcode the index number.

---

### Variation D — `addrs[]` array size must be `NDIRECT + 2`, not `NDIRECT + 1`

Before Task 1:
```c
uint addrs[NDIRECT+1];  // 12+1 = 13 entries: indices 0..12
//                          slot 12 = singly-indirect
```

After Task 1:
```c
uint addrs[NDIRECT+2];  // 11+2 = 13 entries: indices 0..12
//                          slot 11 = singly-indirect
//                          slot 12 = doubly-indirect
```

The **total array size stays 13** — we just rebalance: lose 1 direct slot, gain 1 doubly-indirect slot. A common mistake is declaring `addrs[NDIRECT+1]` after changing `NDIRECT` to 11 — this gives only 12 entries (indices 0..11), dropping the doubly-indirect slot entirely.

---

### Variation E — Checking `addrs[]` size is consistent: `dinode` vs `inode`

Quick verification steps after any NDIRECT change:

```bash
# Check dinode in kernel/fs.h
grep "addrs" kernel/fs.h
# Should show: uint addrs[NDIRECT+2];

# Check inode in kernel/file.h
grep "addrs" kernel/file.h
# Should show: uint addrs[NDIRECT+2];

# Verify NDIRECT value
grep "NDIRECT" kernel/fs.h
# Should show: #define NDIRECT 11
```

Both files must show `NDIRECT+2`. If either shows `NDIRECT+1`, the arrays are out of sync and the file system is broken.

---

### Variation F — `sizeof(struct dinode)` before and after Task 1

Before Task 1:
```
struct dinode:
  short type    = 2 bytes
  short major   = 2 bytes
  short minor   = 2 bytes
  short nlink   = 2 bytes
  uint  size    = 4 bytes
  uint  addrs[13] = 13 × 4 = 52 bytes
Total = 2+2+2+2+4+52 = 64 bytes
```

After Task 1:
```
addrs[NDIRECT+2] = addrs[13] = 13 × 4 = 52 bytes  ← same!
Total = 64 bytes  ← unchanged
```

The struct stays the same size because we go from `addrs[12+1]` (13 entries) to `addrs[11+2]` (13 entries). The on-disk inode layout is identical in byte size — only what each slot *means* changes.

> **Key insight:** Changing `NDIRECT` from 12 to 11 does not change `sizeof(struct dinode)`. This is why the inode blocks count (13 inode blocks) stays the same in the `nmeta` output before and after Task 1.

---

## Quick Reference Cheat Sheet

| Concept | Key value / formula |
|---|---|
| NDIRECT (after) | 11 |
| NINDIRECT | 256 (= 1024 / 4) |
| NDINDIRECT | 256 × 256 = 65,536 |
| MAXFILE | 11 + 256 + 65,536 = **65,803** |
| Direct range | blocks 0 – 10 |
| Singly-indirect range | blocks 11 – 266 |
| Doubly-indirect range | blocks 267 – 65,802 |
| Singly-indirect slot | `addrs[NDIRECT]` = `addrs[11]` |
| Doubly-indirect slot | `addrs[NDIRECT+1]` = `addrs[12]` |
| `addrs[]` total entries | `NDIRECT + 2` = **13** |
| `sizeof(struct dinode)` | **64 bytes** (unchanged before/after Task 1) |
| `index1` formula | `(bn - 267) / 256` |
| `index2` formula | `(bn - 267) % 256` |
| balloc() returns 0 | Disk full — must check and handle |
| balloc() block content | Always zeroed |
| bfree() block content | NOT zeroed — old data remains |
| log_write before brelse | Always — mark dirty before releasing |
| begin_op / end_op | Must be balanced on ALL code paths |
| argstr() before begin_op | Cleaner — no transaction open on arg failure |
| writei user_src = 0 | Kernel buffer source |
| readi user_dst = 0 | Kernel buffer destination |
| namei returns 0 | File not found — return -1, no panic |
| ilock before readi | Always — must hold lock to read inode data |
| iunlock before namei | Always — prevents deadlock |
| iput after iunlock | Always — prevents ref count leak |
| Max symlink depth | 10 |
| O_NOFOLLOW | 0x800 |
| T_SYMLINK | 4 |
| nmeta (FSSIZE=2000) | 46 |
| nmeta (FSSIZE=200000) | 70 |
