---
layout: page
title: "Lab Predictions - lab3-t5r9"
lab: lab3
description: "Exam predictions for Lab3-w8: quiz-style variations on bmap doubly-indirect, itrunc cleanup, symlink system call, and O_NOFOLLOW open() traversal."
---

# ICT1012  
# OPERATING SYSTEMS  
# LABORATORY INSTRUCTIONS  
# Quiz — Lab3-w8 Predictions

---

## Contents

- [Variation 1 — `bmap()` Doubly-Indirect Allocation (7 marks)](#variation-1)
- [Variation 2 — `itrunc()` Doubly-Indirect Cleanup (7 marks)](#variation-2)
- [Variation 3 — `symlink()` System Call (7 marks)](#variation-3)
- [Variation 4 — `open()` Symlink Traversal with Loop Guard (7 marks)](#variation-4)

---

<a name="variation-1"></a>
## Variation 1 — `bmap()` Doubly-Indirect Allocation (7 marks)

### Prerequisite

Download the lab zip from `xSiTe/Labs/xv6labs-w8: FS`. The relevant file is `kernel/fs.c`.

---

### Context

1. You are given the xv6 operating system with a partially implemented file system. The inode layout has been updated to support doubly-indirect blocks as follows:

   - `addrs[0..10]` — 11 direct blocks  
   - `addrs[11]` — singly-indirect block (256 entries)  
   - `addrs[12]` — doubly-indirect block (256 × 256 = 65,536 entries)

2. The function `bmap(struct inode *ip, uint bn)` in `kernel/fs.c` translates a logical block number `bn` into a physical disk block address, allocating new blocks as needed.

3. The direct and singly-indirect cases are already implemented. The doubly-indirect case is **not yet implemented**.

---

### Task Requirements

**Implement the doubly-indirect block allocation inside `bmap()` in `kernel/fs.c`.**

The provided code contains empty placeholders after the singly-indirect block. Complete them with the correct logic.

Your implementation must:

1. **Check range** — enter the doubly-indirect path only when `bn` is beyond the direct and singly-indirect limits.

2. **Level 1 — Master Map:**
   - Read `ip->addrs[NDIRECT+1]`. If it is `0`, call `balloc()` to create the Master Map block and save it to the inode.
   - Call `bread()` to load the Master Map into a buffer.

3. **Calculate indices:**
   - `index1 = bn / NINDIRECT` — which Secondary Map Book to use.
   - `index2 = bn % NINDIRECT` — which slot inside that Secondary Map Book.

4. **Level 2 — Secondary Map:**
   - Check `Master Map[index1]`. If it is `0`, call `balloc()` to allocate a new Secondary Map block, then call `log_write()` to mark the Master Map buffer dirty.
   - Call `brelse()` on the Master Map buffer **before** loading the Secondary Map.
   - Call `bread()` to load the Secondary Map.

5. **Level 3 — Data Block:**
   - Check `Secondary Map[index2]`. If it is `0`, call `balloc()` to allocate the final data block, then call `log_write()` to mark the Secondary Map buffer dirty.
   - Call `brelse()` on the Secondary Map buffer.

6. **Return** the physical data block address.

> **Critical rule:** `brelse()` must be called on every `bread()` buffer as soon as you have finished reading it. Failing to do so will exhaust the buffer cache and panic the kernel.

---

### Task Testing

Run the following to verify your implementation:

```
$ make clean && make qemu
xv6 kernel is booting
init: starting sh
$ bigfile
.................................................................
wrote 6580 blocks
bigfile done; ok
$
```

**Success:** 65 dots printed, followed by `wrote 6580 blocks` and `bigfile done; ok`.

Then run the grading script:

```
$ ./grade-lab-fs bigfile
make: 'kernel/kernel' is up to date.
== Test running bigfile == running bigfile: OK (53.1s)
$
```

---

### Submission

1. Run `make clean && make grade` to confirm output shows `Score: 100/100`.
2. Run `make zipball` to create `lab.zip`.
3. Upload `lab.zip` to Gradescope assignment `xv6labs-w8: fs`.

> **Note:** The grade script does not cover all edge cases. It is your responsibility to carry out further testing, including partial block allocation (e.g., a file that only fills 2 out of 256 Secondary Map Books).

---

<a name="variation-2"></a>
## Variation 2 — `itrunc()` Doubly-Indirect Cleanup (7 marks)

### Prerequisite

Same lab zip as Variation 1. The relevant file is `kernel/fs.c`.

---

### Context

1. When a file is deleted in xv6, the function `itrunc(struct inode *ip)` is responsible for freeing **every** disk block the file occupied and returning them to the free pool.

2. The existing `itrunc()` already handles direct blocks and singly-indirect blocks. It does **not** handle doubly-indirect blocks.

3. If doubly-indirect blocks are not freed, the disk leaks blocks permanently — the file system will fill up over time even though files appear deleted.

---

### Task Requirements

**Extend `itrunc()` in `kernel/fs.c` to free all doubly-indirect blocks.**

Your implementation must, in order:

1. **Load the Master Map:**  
   Check `ip->addrs[NDIRECT+1]`. If it is non-zero, call `bread()` to load it.

2. **Outer loop — iterate over all 256 Master Map entries:**
   - For each non-zero entry at `Master Map[i]`:
     - Call `bread()` to load that Secondary Map block.
     - **Inner loop — iterate over all 256 Secondary Map entries:**
       - For each non-zero entry at `Secondary Map[j]`:
         - Call `bfree()` to free the data block.
     - Call `brelse()` on the Secondary Map buffer.
     - Call `bfree()` on the Secondary Map block address itself.

3. **Free the Master Map:**
   - Call `brelse()` on the Master Map buffer.
   - Call `bfree()` on the Master Map block address.

4. **Clear the inode slot:**  
   Set `ip->addrs[NDIRECT+1] = 0`.

---

### Task Testing

After implementing, run `bigfile` then immediately delete the file and check that disk space is reclaimed:

```
$ make clean && make qemu
xv6 kernel is booting
init: starting sh
$ bigfile
.................................................................
wrote 6580 blocks
bigfile done; ok
$ rm big.file
$ bigfile
.................................................................
wrote 6580 blocks
bigfile done; ok
$
```

**Success:** Running `bigfile` a second time after deleting `big.file` should succeed. If `itrunc()` has a leak, the second run will fail with `write error` because the disk runs out of free blocks.

Run the grading script:

```
$ ./grade-lab-fs bigfile
== Test running bigfile == running bigfile: OK (53.1s)
$
```

---

### Submission

Same as Variation 1.

> **Note:** A common mistake is calling `brelse()` *after* `bfree()`. The correct order is always: finish reading → `brelse()` the buffer → then `bfree()` the block number. Reversing these can corrupt the buffer cache.

---

<a name="variation-3"></a>
## Variation 3 — `symlink()` System Call (7 marks)

### Prerequisite

Same lab zip. Relevant files: `kernel/sysfile.c`, `kernel/stat.h`, `kernel/fcntl.h`, `kernel/syscall.h`, `kernel/syscall.c`, `user/user.h`, `user/usys.pl`.

---

### Context

1. xv6 currently has no support for symbolic links. A symbolic link is a special file of type `T_SYMLINK` whose data blocks store a path string pointing to another file.

2. You are given empty function shells in `kernel/sysfile.c` for `sys_symlink(void)`.

3. The test program `user/symlinktest.c` is provided and will be used to verify your implementation.

---

### Task Requirements

This task has two sub-tasks.

#### Sub-task A — Register the new system call

Make the following additions across the codebase so the kernel recognises `symlink` as a valid system call:

| File | What to add |
|---|---|
| `kernel/stat.h` | `#define T_SYMLINK 4` |
| `kernel/fcntl.h` | `#define O_NOFOLLOW 0x800` |
| `kernel/syscall.h` | `#define SYS_symlink <next_number>` |
| `user/user.h` | `int symlink(const char*, const char*);` |
| `user/usys.pl` | `entry("symlink");` |
| `kernel/syscall.c` | `extern uint64 sys_symlink(void);` and add `[SYS_symlink] sys_symlink,` to the dispatch table |
| `Makefile` | Add `$U/_symlinktest\` to the `UPROGS` list |

> **Flag overlap rule:** `O_NOFOLLOW` must be `0x800`. It must not share any bits with `O_RDONLY (0x000)`, `O_WRONLY (0x001)`, `O_RDWR (0x002)`, `O_CREATE (0x200)`, or `O_TRUNC (0x400)`.

#### Sub-task B — Implement `sys_symlink()` in `kernel/sysfile.c`

Complete the provided empty shell. Your implementation must:

1. Use `argstr(0, target, ...)` and `argstr(1, path, ...)` to fetch the two string arguments from user space.
2. Call `begin_op()` to start a file system transaction.
3. Call `create(path, T_SYMLINK, 0, 0)` to create a new inode of type `T_SYMLINK` at the given path. If this fails, call `end_op()` and return `−1`.
4. Call `writei(ip, 0, (uint64)target, 0, strlen(target))` to write the target path string into the new inode's data blocks.
5. Call `iunlockput(ip)` to release and put the inode.
6. Call `end_op()` to commit the transaction.
7. Return `0` on success.

---

### Task Testing

```
$ make clean && make qemu
xv6 kernel is booting
init: starting sh
$ symlinktest
Start: test symlinks
test symlinks: ok
Start: test concurrent symlinks
test concurrent symlinks: ok
$
```

Run the grading script:

```
$ ./grade-lab-fs symlink
== Test symlinktest: symlinks == symlinktest: symlinks: OK
== Test symlinktest: concurrent symlinks == symlinktest: concurrent symlinks: OK
$
```

---

### Submission

Same as Variation 1.

> **Note:** The grade script does not test dangling symlinks (pointing to a deleted file) or symlinks that point to directories. Carry out these tests manually.

---

<a name="variation-4"></a>
## Variation 4 — `open()` Symlink Traversal with Loop Guard (7 marks)

### Prerequisite

Same lab zip, **and** Sub-task A of Variation 3 must already be completed (types and flags registered). The relevant file is `kernel/sysfile.c`.

---

### Context

1. Even after `sys_symlink()` correctly creates symlink files, the kernel's `sys_open()` function currently opens them **as raw files** — it does not follow the link to the target.

2. You must modify `sys_open()` in `kernel/sysfile.c` so that when it encounters a file of type `T_SYMLINK` and the `O_NOFOLLOW` flag is **not** set, it reads the stored path and redirects to the actual target file.

3. The traversal must be protected against infinite loops (e.g., symlink A → B → A).

---

### Task Requirements

**Modify `sys_open()` in `kernel/sysfile.c` to follow symbolic links.**

Insert the following logic **after the initial `ilock(ip)` call but before the directory-type check** (`if(ip->type == T_DIR ...)`):

1. **Enter a loop** that runs while `ip->type == T_SYMLINK` and `O_NOFOLLOW` is not set in the flags.

2. **Depth guard:** Maintain a counter. If it exceeds `10`, call `iunlockput(ip)` and return `−1` (recursive link error).

3. **Read the target path:**  
   Use `readi(ip, 0, (uint64)path_buf, 0, MAXPATH)` to read the stored target path from the symlink's data blocks into a local buffer.

4. **Unlock and release the current inode:**  
   Call `iunlock(ip)` then `iput(ip)` **before** calling `namei()`. This avoids a deadlock if the target path resolves to the same inode.

5. **Find the next inode:**  
   Call `ip = namei(path_buf)`. If it returns `0`, return `−1` (dangling link — target does not exist).

6. **Lock the new inode:**  
   Call `ilock(ip)` and increment the depth counter. Loop back to step 1.

7. When the loop exits (because `ip->type != T_SYMLINK` or `O_NOFOLLOW` is set), continue with the normal `sys_open()` flow.

---

### Locking Rules (Critical)

| Rule | Reason |
|---|---|
| Always `iunlock(ip)` before `namei()` | Prevents deadlock if the chain loops back to the same inode |
| Always `ilock(ip)` before `readi()` | You must hold the lock to safely read inode data |
| `iput(ip)` after `iunlock(ip)` | Drops the reference count; prevents inode memory leak |
| Never call `namei()` while holding a lock | `namei()` may call `ilock()` internally; double-locking panics |

---

### Task Testing

```
$ make clean && make qemu
xv6 kernel is booting
init: starting sh
$ symlinktest
Start: test symlinks
test symlinks: ok
Start: test concurrent symlinks
test concurrent symlinks: ok
$
```

Run the full grading script:

```
$ make clean && make grade
== Test running bigfile ==   running bigfile: OK (91.2s)
== Test running symlinktest ==   (2.0s)
== Test symlinktest: symlinks ==   symlinktest: symlinks: OK
== Test symlinktest: concurrent symlinks ==   symlinktest: concurrent symlinks: OK
Score: 100/100
$
```

---

### Submission

1. Run `make clean && make grade` and confirm `Score: 100/100`.
2. Run `make zipball` to create `lab.zip`.
3. Upload `lab.zip` to Gradescope assignment `xv6labs-w8: fs`.

> **Note:** The grade script does not test the boundary case of exactly 10 chained symlinks or a symlink pointing to a directory. Test these manually before submitting.

---

## Quick Reference

| Concept | Value |
|---|---|
| NDIRECT (after Task 1) | 11 |
| NINDIRECT | 256 |
| Doubly-indirect start | block 267 |
| `index1` formula | `(bn - 267) / 256` |
| `index2` formula | `(bn - 267) % 256` |
| MAXFILE | 65,803 |
| `T_SYMLINK` value | `4` |
| `O_NOFOLLOW` flag | `0x800` |
| Max symlink depth | 10 |
| `addrs[]` array size | `NDIRECT + 2` = 13 |
