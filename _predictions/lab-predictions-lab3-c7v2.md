---
layout: page
title: "Lab Predictions - lab3-c7v2"
lab: lab3
description: "Exam predictions for Lab3-w8: T_SYMLINK vs T_FILE vs T_DIR type checking, concurrent symlink race conditions, itrunc order of operations, bmap allocation failure recovery, create() internals, and FSSIZE vs MAXFILE independence."
---

# Lab3-w8 Exam Predictions (New Angles)

---

## Topic 1 — File Type Checking: `T_SYMLINK` vs `T_FILE` vs `T_DIR`

xv6 defines four file types in `kernel/stat.h`. After Task 2, `T_SYMLINK` is added as the fourth type. Every inode has a `type` field that determines how the kernel handles it.

| Constant | Value | Meaning |
|---|---|---|
| `T_DIR` | 1 | Directory |
| `T_FILE` | 2 | Regular file |
| `T_DEVICE` | 3 | Device file |
| `T_SYMLINK` | 4 | Symbolic link |

---

### Variation A — Where `ip->type` is checked in `sys_open()`

The existing `sys_open()` code checks for `T_DIR` to prevent opening directories in write mode:

```c
if(ip->type == T_DIR && omode != O_RDONLY){
    iunlockput(ip);
    end_op();
    return -1;
}
```

The new symlink check must be inserted **before** this directory check. If it is inserted after, a symlink pointing to a directory passes through the symlink loop (unresolved) and then hits the directory check — wrong behaviour.

> **Correct insertion point:** after `ilock(ip)`, before `if(ip->type == T_DIR ...)`.

---

### Variation B — Type check inside the symlink-following loop

```c
while(ip->type == T_SYMLINK && !(omode & O_NOFOLLOW)){
    if(depth++ >= 10){
        iunlockput(ip);
        end_op();
        return -1;
    }
    // readi, iunlockput, namei, ilock...
}
// After loop: ip->type is guaranteed to NOT be T_SYMLINK
// (or O_NOFOLLOW was set)
```

When the loop exits normally, `ip->type` can be `T_FILE`, `T_DIR`, or `T_DEVICE`. The existing code then handles each correctly:
- `T_DIR` — blocked if not `O_RDONLY`
- `T_FILE` — normal file open
- `T_DEVICE` — device open

---

### Variation C — What if `T_SYMLINK` is assigned value 2 (same as `T_FILE`)?

```c
#define T_SYMLINK 2   // WRONG — clashes with T_FILE
```

The kernel cannot distinguish between regular files and symlinks. Every regular file would be treated as a symlink in `sys_open()` — the loop would try to `readi()` the file's data as a path string and call `namei()` on garbage. **All file opens would fail or return wrong files.**

> **Why 4?** It is the next unused integer after `T_DEVICE = 3`. Any value ≥ 4 that is not already used would work, but 4 is the natural choice.

---

### Variation D — `create()` sets the type field

```c
ip = create(path, T_SYMLINK, 0, 0);
```

`create()` allocates a new inode using `ialloc()` and sets `ip->type = T_SYMLINK`. The third and fourth arguments (major, minor) are device numbers used only for `T_DEVICE` — they are 0 for symlinks.

After `create()` returns, the inode is:
- Locked (`ilock` was called internally)
- Type set to `T_SYMLINK`
- `nlink = 1` (one directory entry points to it)
- Data blocks: all zeros (empty — no target path written yet)

`writei()` then stores the target path in the data blocks.

---

### Variation E — Reading `ip->type` without `ilock()` (race condition)

```c
ip = namei(path);
// NO ilock here
if(ip->type == T_SYMLINK) ...   // ← RACE CONDITION
```

Another process could be calling `create()` on a new file with the same inode number simultaneously, writing `ip->type` at the exact same time this code reads it. The read could see a partially written value. Always call `ilock(ip)` before reading any inode field.

---

## Topic 2 — Concurrent Symlink Creation (`test concurrent symlinks`)

The `symlinktest` concurrent test spawns multiple child processes that simultaneously create symlinks and open them. This stresses the locking logic in `sys_symlink()` and `sys_open()`.

---

### Variation A — What the concurrent test checks

Multiple processes each:
1. Create a file `"a_N"` (where N is process number).
2. Create a symlink `"b_N"` pointing to `"a_N"`.
3. Open `"b_N"` and verify the file descriptor is valid.
4. Verify reading from the fd returns the expected data from `"a_N"`.

If any process gets a wrong result or panics, the test fails. The test is specifically checking:
- `create()` is safe under concurrent calls (protected by inode locks and the log).
- `sys_open()` following symlinks does not cause data races.

---

### Variation B — Why `ilock()`/`iunlock()` prevents race conditions in `sys_open()`

```c
ilock(ip);
// ip->type, ip->size, ip->addrs[] are now safe to read
readi(ip, 0, (uint64)path_buf, 0, MAXPATH);
iunlock(ip);
```

Between `ilock()` and `iunlock()`, no other process can modify `ip`. Without the lock:
- Process A reads `ip->type` as `T_SYMLINK`
- Process B sets `ip->type = T_FILE` (e.g., by truncating and repurposing the inode)
- Process A reads stale data as a path — wrong result

---

### Variation C — Why the log (`begin_op`/`end_op`) is needed for concurrent creates

If two processes both call `create()` simultaneously without the log:
- Both might call `ialloc()` and get the same inode number (if the bitmap write is not atomic).
- Both write to the same inode — corruption.

`begin_op()` / `end_op()` ensures each `create()` is an atomic transaction. The log serialises conflicting writes — one transaction commits fully before the other begins its disk writes.

---

### Variation D — What "test concurrent symlinks: ok" proves

The test passing proves:
1. No kernel panic under concurrent symlink creation and traversal.
2. No deadlock — all processes eventually complete.
3. No data race — each process reads back the correct file data through its own symlink.
4. Inode reference counts (`ip->ref`) are balanced — no inode table exhaustion.

A single missing `iput()` or `brelse()` that causes a leak may not be caught by the basic symlinks test but will manifest under the concurrent test (which runs many more operations).

---

## Topic 3 — `itrunc()` Order of Operations

`itrunc()` must free blocks in a specific order. The wrong order either leaks blocks, reads freed memory, or panics the kernel.

---

### Variation A — Correct top-level order in `itrunc()`

```c
void itrunc(struct inode *ip){
  // 1. Free direct blocks
  for(int i = 0; i < NDIRECT; i++){
    if(ip->addrs[i]){
      bfree(ip->dev, ip->addrs[i]);
      ip->addrs[i] = 0;
    }
  }

  // 2. Free singly-indirect blocks
  if(ip->addrs[NDIRECT]){
    bp = bread(ip->dev, ip->addrs[NDIRECT]);
    // free 256 data blocks, then brelse, then bfree the map
    ...
    ip->addrs[NDIRECT] = 0;
  }

  // 3. Free doubly-indirect blocks
  if(ip->addrs[NDIRECT+1]){
    // free all secondary maps and their data blocks, then master map
    ...
    ip->addrs[NDIRECT+1] = 0;
  }

  ip->size = 0;
  iupdate(ip);
}
```

The tiers are freed in order: direct → singly-indirect → doubly-indirect. Within each tier, data blocks are freed before the map blocks that point to them (but only after their containing buffer is `brelse()`d).

---

### Variation B — Wrong: clearing `ip->addrs[]` before `bfree()`

```c
uint addr = ip->addrs[NDIRECT+1];
ip->addrs[NDIRECT+1] = 0;       // ← cleared too early
// ...
bfree(ip->dev, addr);           // uses saved addr — OK actually
```

Clearing the inode slot before `bfree()` is technically safe (we saved `addr` to a local variable). But clearing it before `brelse()` of related buffers can cause confusion. The recommended pattern is: finish all `brelse()` calls → call `bfree()` → then clear the slot.

---

### Variation C — Wrong: `bfree()` before `brelse()`

```c
// Inside doubly-indirect cleanup:
bfree(ip->dev, master[i]);       // ← free the Secondary Map disk block
brelse(sec_bp);                  // ← now releasing buffer for a freed block
```

`bfree()` marks the block as free in the bitmap. Another process running `balloc()` simultaneously could reuse that block immediately. Then `brelse(sec_bp)` releases a buffer that another process now owns — potentially corrupting that other process's data.

**Correct order:**
```c
brelse(sec_bp);                  // release in-memory buffer first
bfree(ip->dev, master[i]);       // then mark disk block as free
```

---

### Variation D — Wrong: not calling `iupdate()` at the end of `itrunc()`

```c
ip->size = 0;
// forgot: iupdate(ip);
```

`iupdate()` writes the updated in-memory inode (with `size = 0` and zeroed `addrs[]`) back to the on-disk dinode. Without this call, the inode on disk still shows the old size and old block addresses — even though those blocks have been freed. After a reboot, `ilock()` reads the old on-disk inode and the kernel thinks the file still exists with its old blocks, which may now be allocated to other files. **Double-allocation corruption.**

---

### Variation E — What `ip->size = 0` alone does NOT do

Setting `ip->size = 0` only updates the in-memory inode's size field. It does NOT:
- Free any disk blocks (`bfree()` is required for that).
- Clear `ip->addrs[]` entries (each must be explicitly set to 0).
- Write anything to disk (`iupdate()` is required for that).

A student who only sets `ip->size = 0` without any `bfree()` calls effectively creates a "ghost file" — the blocks are still allocated in the bitmap but the inode no longer knows where they are. The disk slowly fills up permanently.

---

## Topic 4 — `bmap()` Allocation Failure Recovery

When `balloc()` returns 0 (disk full), `bmap()` must not leave partially allocated structures behind.

---

### Variation A — Failure after Master Map allocated, before Secondary Map allocated

```c
// Master Map just allocated:
addr = balloc(dev);     // → 500 (Master Map)
ip->addrs[NDIRECT+1] = 500;
bp = bread(dev, 500);
a = (uint*)bp->data;

// Secondary Map allocation fails:
addr = balloc(dev);     // → 0 (disk full!)
if(addr == 0){
    brelse(bp);         // ← must release Master Map buffer
    return 0;
}
```

The Master Map block (500) is allocated and saved to the inode. On return 0, `bmap()` exits without allocating the Secondary Map. The Master Map block is "stranded" — allocated in the bitmap, written to `addrs[12]`, but currently empty (all zeros). This is **acceptable** — the next call to `bmap()` for a block in this zone will find the existing Master Map and try again.

> **Not a leak:** The Master Map will be freed by `itrunc()` when the file is deleted. Returning 0 here just means this particular write fails, not that the block is permanently lost.

---

### Variation B — Failure after Secondary Map allocated, before data block allocated

```c
// Secondary Map just allocated:
a[index1] = balloc(dev);    // → 501 (Secondary Map)
log_write(bp);
brelse(bp);
bp = bread(dev, 501);
b = (uint*)bp->data;

// Data block allocation fails:
addr = balloc(dev);     // → 0 (disk full!)
if(addr == 0){
    brelse(bp);         // ← must release Secondary Map buffer
    return 0;
}
```

Same situation — Secondary Map is allocated and linked from Master Map, but the data block is not. On the next `bmap()` call for this block, the Secondary Map already exists and will be reused.

---

### Variation C — What happens if `brelse()` is missing on failure path

```c
if(addr == 0){
    // forgot brelse(bp)
    return 0;
}
```

The buffer for the Master Map (or Secondary Map) is permanently locked. After enough such failures, all 30 buffer cache slots fill up. The next `bread()` call hangs forever. The system appears to freeze with no error message.

> **Rule:** Every failure path in `bmap()` that exits early must release every buffer it has `bread()`ed before returning.

---

### Variation D — Failure in `balloc()` for the Master Map itself

```c
addr = balloc(dev);     // → 0 (disk full, couldn't even get a Master Map block)
if(addr == 0)
    return 0;
```

This is the cleanest failure — nothing was allocated, nothing was loaded into the buffer cache, nothing needs cleanup. Just return 0.

---

## Topic 5 — FSSIZE vs MAXFILE Independence

A common misconception is that increasing `FSSIZE` automatically allows larger files, or that increasing `MAXFILE` automatically gives more disk space.

---

### Variation A — FSSIZE controls disk capacity, MAXFILE controls file size limit

| Parameter | Controls | Location |
|---|---|---|
| `FSSIZE` | Total number of disk blocks in `fs.img` | `kernel/param.h` |
| `MAXFILE` | Maximum blocks any single file can use | `kernel/fs.h` |

These are completely independent. You could have `FSSIZE = 200000` (large disk) but `MAXFILE = 268` (small max file) — the disk has plenty of space but no single file can be large. You could also have `FSSIZE = 2000` (small disk) but `MAXFILE = 65803` — a single file could theoretically use 65803 blocks, but the disk only has ~1954 data blocks total, so `bmap()` would hit `balloc()` returning 0 long before reaching the theoretical max.

---

### Variation B — `bigfile` fails even after MAXFILE is increased, if FSSIZE is not increased

```
$ bigfile
..
write error
bigfile: file is too small
```

`bigfile` tries to write 6580 blocks. With `FSSIZE = 2000`, there are only ~1954 data blocks available total. Even though `MAXFILE = 65803` allows files that large in theory, the physical disk runs out of space. `balloc()` returns 0, `bmap()` returns 0, `writei()` returns an error.

**Both changes are required:** `FSSIZE → 200000` AND `NDIRECT → 11` with `MAXFILE → 65803`.

---

### Variation C — Can MAXFILE exceed available data blocks in FSSIZE?

Yes — and it is fine. `MAXFILE` is a theoretical ceiling based on inode structure. The actual limit is `min(MAXFILE, available_data_blocks)`. A file system with `FSSIZE = 10000` and `MAXFILE = 65803` can hold files up to approximately 9930 blocks (10000 minus metadata), not 65803.

---

### Variation D — FSSIZE = 200000 but NDIRECT not changed: what fails?

If `FSSIZE = 200000` but `NDIRECT` stays at 12 (no doubly-indirect support):

- `bigfile` tries to write block 269 (which is beyond 268 = old MAXFILE).
- `bmap()` hits the `panic("bmap: out of range")` assertion.
- The kernel panics — xv6 crashes.

The large disk has plenty of space, but the inode cannot address beyond 268 blocks. More disk space without a larger inode structure is useless.

---

### Variation E — Relationship between FSSIZE and bitmap blocks (nmeta dependency)

Increasing `FSSIZE` from 2000 to 200000 increases bitmap blocks from 1 to 25 (each bitmap block tracks 8192 disk blocks). These 24 extra bitmap blocks come out of `FSSIZE` — they reduce the data block count.

```
FSSIZE = 200000
bitmap_blocks = ceil(200000 / 8192) = 25
nmeta = 1+1+30+13+25 = 70
data_blocks = 200000 - 70 = 199930
```

The 24 extra bitmap blocks reduce data blocks by 24 (from the perspective of `nmeta`). This is negligible compared to the 197976 extra blocks gained, but the exam may ask you to calculate the exact data block count.

---

## Topic 6 — Step-by-Step `sys_open()` Full Trace for a Symlink

Trace every kernel action when `open("link", O_RDONLY)` is called, where `link → real_file`.

---

### Variation A — Full kernel trace (happy path)

```
User calls: open("link", O_RDONLY)
                ↓
sys_open() begins
  argstr(0, path) → path = "link"
  argint(1, &omode) → omode = O_RDONLY = 0x000
  begin_op()

  ip = namei("link")          → finds link inode (T_SYMLINK)
  ilock(ip)                   → link inode now locked

  --- symlink loop ---
  depth = 0
  ip->type == T_SYMLINK && !(omode & O_NOFOLLOW) → true, enter loop
    depth++ → depth = 1
    1 <= 10 → continue
    readi(ip, 0, path_buf, 0, MAXPATH) → path_buf = "real_file"
    iunlock(ip)
    iput(ip)                  → link ref=0
    ip = namei("real_file")   → finds real_file inode (T_FILE)
    ilock(ip)                 → real_file inode locked
  ip->type == T_FILE → loop exits

  --- normal open continues ---
  if(ip->type == T_DIR && omode != O_RDONLY) → false, skip
  f = filealloc()             → allocate file struct
  fd = fdalloc(f)             → allocate file descriptor
  f->type = FD_INODE
  f->ip = ip
  f->readable = 1
  iunlock(ip)                 → real_file inode unlocked
  end_op()

  return fd
```

---

### Variation B — Full kernel trace (circular symlink, depth limit hit)

```
open("a", O_RDONLY) where a→b, b→a

  ip = namei("a"), ilock(ip)

  Loop iteration 1: depth=1, read "b", iunlockput(a), ip=namei("b"), ilock(b)
  Loop iteration 2: depth=2, read "a", iunlockput(b), ip=namei("a"), ilock(a)
  Loop iteration 3: depth=3, read "b", iunlockput(a), ip=namei("b"), ilock(b)
  ...
  Loop iteration 10: depth=10, guard fires:
    iunlockput(ip)
    end_op()
    return -1
```

User space receives `fd = -1`. No panic.

---

### Variation C — Full kernel trace (`O_NOFOLLOW` set)

```
open("link", O_RDONLY | O_NOFOLLOW)

  ip = namei("link"), ilock(ip)

  ip->type == T_SYMLINK → true
  !(omode & O_NOFOLLOW) → !(0x800) → false
  → loop condition false, loop NOT entered

  ip->type == T_DIR? → No
  filealloc(), fdalloc()
  f->ip = ip  (points to the symlink inode itself)
  iunlock(ip)
  end_op()
  return fd  (fd for the symlink file, not the target)
```

Reading from this fd returns the raw bytes stored in the symlink (the target path string).

---

### Variation D — Full kernel trace (dangling link)

```
open("link", O_RDONLY) where link → "deleted_file" (does not exist)

  ip = namei("link"), ilock(ip)
  Loop: depth=1, read "deleted_file", iunlockput(link)
  ip = namei("deleted_file") → returns 0 (not found)

  if(ip == 0):
    end_op()
    return -1
```

User space receives `fd = -1`.

---

## Quick Reference Cheat Sheet

| Concept | Key value / formula |
|---|---|
| NDIRECT (after) | 11 |
| NINDIRECT | 256 |
| NDINDIRECT | 65,536 |
| MAXFILE | **65,803** |
| `T_DIR` | 1 |
| `T_FILE` | 2 |
| `T_DEVICE` | 3 |
| `T_SYMLINK` | 4 |
| `O_NOFOLLOW` | 0x800 |
| Symlink loop condition | `ip->type == T_SYMLINK && !(omode & O_NOFOLLOW)` |
| Symlink loop insertion point | After `ilock(ip)`, before `T_DIR` check |
| Max depth | 10 |
| On depth hit | `iunlockput(ip)`, `end_op()`, return −1 |
| On dangling link | `namei()` returns 0 → `end_op()`, return −1 |
| On `O_NOFOLLOW` | Loop condition false → open symlink itself |
| `brelse` vs `bfree` order | `brelse` first, then `bfree` |
| `log_write` vs `brelse` order | `log_write` first, then `brelse` |
| `begin_op`/`end_op` balance | Every code path must call `end_op` |
| `argstr` timing | Before `begin_op` — cleaner error handling |
| `iupdate()` at end of `itrunc` | Required — writes zeroed inode back to disk |
| `ip->size = 0` alone | NOT enough — does not free blocks or write to disk |
| `create()` returns inode | Locked, type set, nlink=1, empty data |
| `FSSIZE` vs `MAXFILE` | Independent — both must be updated |
| Bitmap block coverage | 8,192 blocks per bitmap block |
| nmeta (FSSIZE=200000) | 70 |
| `sizeof(struct dinode)` | 64 bytes (unchanged before/after Task 1) |
