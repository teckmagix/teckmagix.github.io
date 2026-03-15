---
layout: page
title: "Lab Predictions - lab3-x9p3"
lab: lab3
description: "Exam predictions for Lab3-w8: kernel struct updates, nmeta calculation task, bmap index fill-in, symlink open() edge cases, and combined Task1+Task2 integration."
---

# ICT1012  
# OPERATING SYSTEMS  
# LABORATORY INSTRUCTIONS  
# Quiz — Lab3-w8 Predictions (Extended)

---

## Contents

- [Variation 5 — Kernel Struct & Header Updates (7 marks)](#variation-5)
- [Variation 6 — `nmeta` Calculation as a Coding/Trace Task (7 marks)](#variation-6)
- [Variation 7 — `bmap()` Index Fill-in-the-Blanks (7 marks)](#variation-7)
- [Variation 8 — `open()` Edge Cases: Dangling, Chained, Circular (7 marks)](#variation-8)

---

<a name="variation-5"></a>
## Variation 5 — Kernel Struct & Header Updates (7 marks)

### Prerequisite

Download the lab zip from `xSiTe/Labs/xv6labs-w8: FS`. The relevant files are `kernel/fs.h`, `kernel/file.h`, and `kernel/param.h`.

---

### Context

1. The original xv6 file system limits files to 268 blocks using 12 direct + 1 singly-indirect inode slot.

2. To support doubly-indirect blocks, the inode layout must change in **both** the on-disk representation (`struct dinode` in `kernel/fs.h`) and the in-memory representation (`struct inode` in `kernel/file.h`).

3. These two structs **must always match** in the number of `addrs[]` entries. A mismatch causes silent data corruption — the kernel writes to the wrong physical block addresses.

4. The disk image size must also increase from 2000 blocks to 200000 blocks in `kernel/param.h` to accommodate larger files.

---

### Task Requirements

Make **all** of the following changes. The code will compile and partially work with incomplete changes, but will fail test cases.

#### Sub-task A — Update `kernel/param.h`

Change the file system size:

```c
// Before
#define FSSIZE 2000

// After
#define FSSIZE 200000
```

This causes `mkfs` to build a larger disk image. Verify the change by checking the `nmeta` line in `make` output:

```
// Before
nmeta 46 ... blocks 1954 total 2000

// After
nmeta 70 ... blocks 199930 total 200000
```

> If the `nmeta` line still shows `total 2000`, the change did not take effect. Run `make clean` to force a rebuild.

#### Sub-task B — Update `kernel/fs.h`

Make the following three changes:

1. Change `NDIRECT` from `12` to `11`:

```c
#define NDIRECT 11
```

2. Define `NDINDIRECT` (the doubly-indirect capacity):

```c
#define NDINDIRECT (NINDIRECT * NINDIRECT)
```

3. Update `MAXFILE` to include all three tiers:

```c
#define MAXFILE (NDIRECT + NINDIRECT + NDINDIRECT)
```

4. Update `struct dinode` so `addrs[]` has room for direct + singly-indirect + doubly-indirect slots:

```c
// Before
uint addrs[NDIRECT+1];

// After
uint addrs[NDIRECT+2];
```

#### Sub-task C — Update `kernel/file.h`

Mirror the same `addrs[]` change in the in-memory inode struct:

```c
// Before
uint addrs[NDIRECT+1];

// After
uint addrs[NDIRECT+2];
```

> **Critical:** `struct inode` (in-memory) and `struct dinode` (on-disk) must always have identical `addrs[]` array sizes. If they differ, `ilock()` will copy the wrong number of addresses from disk into memory, corrupting every file access.

#### Sub-task D — Rebuild the disk image

After changing `NDIRECT`, the existing `fs.img` is invalid. Delete it and rebuild:

```
$ rm fs.img
$ make clean && make qemu
```

If `fs.img` is not deleted, xv6 will boot with a corrupted file system that was built with the old `NDIRECT=12` layout.

---

### Task Testing

After all changes, confirm the `nmeta` line in `make` output:

```
nmeta 70 (boot, super, log blocks 30 inode blocks 13, bitmap blocks 25) blocks 199930 total 200000
```

Then run `bigfile` to confirm the full system works end to end:

```
$ bigfile
.................................................................
wrote 6580 blocks
bigfile done; ok
$
```

---

### Submission

1. Run `make clean && make grade` and confirm `Score: 100/100`.
2. Run `make zipball` to create `lab.zip`.
3. Upload `lab.zip` to Gradescope assignment `xv6labs-w8: fs`.

> **Note:** The most common mistake in this variation is updating only one of the two structs. Always check both `kernel/fs.h` (dinode) and `kernel/file.h` (inode) together.

---

<a name="variation-6"></a>
## Variation 6 — `nmeta` Calculation as a Coding/Trace Task (7 marks)

### Prerequisite

Same lab zip. The relevant file is `kernel/param.h`. No code changes are required — this task tests your ability to **trace and predict** the output of `mkfs`.

---

### Context

1. When xv6 boots, `mkfs/mkfs.c` builds the disk image and prints a single `nmeta` line describing the layout.

2. `nmeta` is the total number of **metadata blocks** — blocks used by the file system itself rather than storing file data.

3. The formula for `nmeta` is:

```
nmeta = 1 (boot)
      + 1 (superblock)
      + LOGSIZE (log blocks, default 30)
      + ceil(NINODES * sizeof(struct dinode) / BSIZE)   (inode blocks)
      + ceil(FSSIZE / (BSIZE * 8))                      (bitmap blocks)
```

Where:
- `BSIZE = 1024` bytes
- `NINODES = 200` (default in xv6)
- `sizeof(struct dinode) = 64` bytes → inode blocks = `ceil(200 * 64 / 1024)` = `ceil(12.5)` = **13**
- Each bitmap block tracks `1024 * 8 = 8192` disk blocks

---

### Task Requirements

For each given `FSSIZE`, calculate and fill in the complete `nmeta` output line.

#### Question 1 — FSSIZE = 2000

| Component | Calculation | Result |
|---|---|---|
| Boot | — | 1 |
| Superblock | — | 1 |
| Log blocks | — | 30 |
| Inode blocks | `ceil(200 × 64 / 1024)` | **13** |
| Bitmap blocks | `ceil(2000 / 8192)` | **?** |
| **nmeta** | sum | **?** |
| Data blocks | `2000 − nmeta` | **?** |

**Expected answer:**

```
nmeta 46 (boot, super, log blocks 30 inode blocks 13, bitmap blocks 1) blocks 1954 total 2000
```

---

#### Question 2 — FSSIZE = 200000

| Component | Calculation | Result |
|---|---|---|
| Boot | — | 1 |
| Superblock | — | 1 |
| Log blocks | — | 30 |
| Inode blocks | `ceil(200 × 64 / 1024)` | **13** |
| Bitmap blocks | `ceil(200000 / 8192)` | **?** |
| **nmeta** | sum | **?** |
| Data blocks | `200000 − nmeta` | **?** |

**Expected answer:**

```
nmeta 70 (boot, super, log blocks 30 inode blocks 13, bitmap blocks 25) blocks 199930 total 200000
```

---

#### Question 3 — FSSIZE = 16384

| Component | Calculation | Result |
|---|---|---|
| Bitmap blocks | `ceil(16384 / 8192)` | **?** |
| **nmeta** | | **?** |
| Data blocks | | **?** |

**Expected answer:**

```
nmeta 48 (boot, super, log blocks 30 inode blocks 13, bitmap blocks 2) blocks 16336 total 16384
```

---

#### Question 4 — FSSIZE = 8192

| Component | Calculation | Result |
|---|---|---|
| Bitmap blocks | `ceil(8192 / 8192)` | **?** |
| **nmeta** | | **?** |
| Data blocks | | **?** |

**Expected answer:**

```
nmeta 46 (boot, super, log blocks 30 inode blocks 13, bitmap blocks 1) blocks 8146 total 8192
```

> **Exam trap:** FSSIZE = 8192 gives the same `nmeta` as FSSIZE = 2000 (both need only 1 bitmap block), but different data block counts. A common mistake is assuming `nmeta` always changes when `FSSIZE` changes.

---

#### Question 5 — FSSIZE = 8193

| Component | Calculation | Result |
|---|---|---|
| Bitmap blocks | `ceil(8193 / 8192)` | **?** |
| **nmeta** | | **?** |
| Data blocks | | **?** |

**Expected answer:**

```
nmeta 47 (boot, super, log blocks 30 inode blocks 13, bitmap blocks 2) blocks 8146 total 8193
```

> **Exam trap:** Adding just 1 block beyond 8192 forces a second bitmap block. The boundary at multiples of 8192 is the key insight here.

---

### Task Testing

Set `FSSIZE` in `kernel/param.h` to each value above, run `make clean && make qemu`, and verify the printed `nmeta` line matches your calculation before the xv6 shell appears.

---

### Submission

Same as Variation 5.

> **Note:** You do not need to run `bigfile` for this variation. The grader checks only the `nmeta` line output during boot.

---

<a name="variation-7"></a>
## Variation 7 — `bmap()` Index Fill-in-the-Blanks (7 marks)

### Prerequisite

Same lab zip. The relevant file is `kernel/fs.c`. This task tests your ability to correctly complete a **partially written** `bmap()` doubly-indirect implementation.

---

### Context

1. The `bmap()` function is provided with several `/* YOUR CODE HERE */` placeholders.

2. You must fill in each placeholder with the single correct expression. Writing the wrong expression — even if it compiles — will silently read or write the wrong physical disk block.

3. After the direct (slots 0–10) and singly-indirect (slot 11) zones are handled, the remaining logical block numbers fall into the doubly-indirect zone. The block number `bn` passed into this section has already been adjusted:

```c
bn -= NINDIRECT;  // subtract singly-indirect count (256)
// at this point bn is relative to start of doubly-indirect zone
```

---

### Task Requirements

Fill in **every** `/* YOUR CODE HERE */` in the following code skeleton. Each blank is worth marks.

```c
// Doubly-indirect block
if(bn < NDINDIRECT){
  // Level 1: load or allocate the Master Map block
  if((addr = ip->addrs[/* BLANK 1 */]) == 0){
    addr = balloc(ip->dev);
    if(addr == 0) return 0;
    ip->addrs[/* BLANK 2 */] = addr;
  }
  bp = bread(ip->dev, addr);
  a = (uint*)bp->data;

  // Calculate index into the Master Map
  index1 = /* BLANK 3 */;

  // Level 2: load or allocate the Secondary Map block
  if((addr = a[index1]) == 0){
    addr = balloc(ip->dev);
    if(addr == 0){
      brelse(bp);
      return 0;
    }
    a[index1] = addr;
    log_write(/* BLANK 4 */);
  }
  brelse(/* BLANK 5 */);
  bp = bread(ip->dev, addr);
  a = (uint*)bp->data;

  // Calculate index into the Secondary Map
  index2 = /* BLANK 6 */;

  // Level 3: load or allocate the final Data Block
  if((addr = a[index2]) == 0){
    addr = balloc(ip->dev);
    if(addr == 0){
      brelse(bp);
      return 0;
    }
    a[index2] = addr;
    log_write(/* BLANK 7 */);
  }
  brelse(/* BLANK 8 */);
  return addr;
}
```

#### Answers

| Blank | Correct expression | Explanation |
|---|---|---|
| BLANK 1 | `NDIRECT + 1` | Slot 12 (0-indexed) holds the doubly-indirect pointer |
| BLANK 2 | `NDIRECT + 1` | Same slot — saving the newly allocated address back |
| BLANK 3 | `bn / NINDIRECT` | Which Secondary Map Book (row in the master map) |
| BLANK 4 | `bp` | Mark the **Master Map** buffer dirty after writing index1 |
| BLANK 5 | `bp` | Release the **Master Map** buffer before loading Secondary Map |
| BLANK 6 | `bn % NINDIRECT` | Which Data Block slot within the Secondary Map |
| BLANK 7 | `bp` | Mark the **Secondary Map** buffer dirty after writing index2 |
| BLANK 8 | `bp` | Release the **Secondary Map** buffer before returning |

> **Common wrong answers:**
> - BLANK 3: Writing `bn / 256` instead of `bn / NINDIRECT` — compiles and works, but is bad style and will break if `BSIZE` changes.
> - BLANK 4 & 5: Swapping `log_write` and `brelse` — `brelse` releases the buffer back to the cache; calling it before `log_write` means you are marking a buffer you no longer own.
> - BLANK 1 & 2: Writing `NDIRECT` instead of `NDIRECT + 1` — this accidentally reads/writes the singly-indirect slot, silently corrupting it.

---

### Task Testing

```
$ make clean && make qemu
$ bigfile
.................................................................
wrote 6580 blocks
bigfile done; ok
$
```

Run the grading script:

```
$ ./grade-lab-fs bigfile
== Test running bigfile == running bigfile: OK (53.1s)
$
```

---

### Submission

Same as Variation 5.

> **Note:** Even a single wrong blank will cause `bigfile` to either panic the kernel or write to wrong disk blocks. If you see a kernel panic mentioning `bget: no buffers`, it means a `brelse()` was missed (Blank 5 or Blank 8).

---

<a name="variation-8"></a>
## Variation 8 — `open()` Edge Cases: Dangling, Chained, Circular (7 marks)

### Prerequisite

Same lab zip with Variations 3 and 4 already complete (symlink type registered, `sys_symlink()` implemented). The relevant file is `kernel/sysfile.c`.

---

### Context

1. A correct `sys_open()` symlink traversal must handle three distinct edge cases beyond the simple "one link to one file" case.

2. Each edge case has a specific expected return value and kernel behaviour. Returning the wrong value or panicking the kernel for any case is a bug.

3. The test program `user/symlinktest.c` exercises all three cases. You must ensure your loop handles them correctly.

---

### Task Requirements

For each edge case below, describe the exact kernel behaviour your implementation must produce, and identify which line of code inside the traversal loop is responsible.

---

#### Sub-task A — Dangling Symlink (target was deleted)

**Setup:**
```c
// In symlinktest.c
int fd = open("real_file", O_CREATE | O_WRONLY);
close(fd);
symlink("real_file", "mylink");
unlink("real_file");           // delete the target
fd = open("mylink", O_RDONLY); // what should happen?
```

**Required behaviour:**
- `namei("real_file")` returns `0` (inode not found).
- Your loop must detect this and return **`−1`** to user space.
- The kernel must **not** panic.

**Responsible code:**
```c
ip = namei(path_buf);
if(ip == 0){
  end_op();
  return -1;   // ← this line handles the dangling case
}
```

---

#### Sub-task B — Chained Symlink (link4 → link3 → link2 → link1 → file)

**Setup:**
```c
// Chain of 4 symlinks
symlink("file",  "link1");
symlink("link1", "link2");
symlink("link2", "link3");
symlink("link3", "link4");
fd = open("link4", O_RDONLY);  // should succeed
```

**Required behaviour:**
- The loop iterates **4 times** (depth 1 → 2 → 3 → 4).
- On the 4th iteration, `ip->type` is no longer `T_SYMLINK` (it is `T_FILE`).
- The loop exits and `open()` returns a valid file descriptor.
- Depth counter after loop exits: **4** — well within the limit of 10.

**What fails here if the depth guard is wrong:**  
If the guard condition is `depth >= 4` instead of `depth > 10`, this legitimate 4-link chain is incorrectly rejected with `−1`.

---

#### Sub-task C — Circular Symlink (A → B → A)

**Setup:**
```c
symlink("b", "a");   // "a" points to "b"
symlink("a", "b");   // "b" points to "a"
fd = open("a", O_RDONLY);  // should return -1
```

**Required behaviour:**
- Loop iteration trace:

| Depth | Current file | `ip->type` | Action |
|---|---|---|---|
| 1 | `a` | `T_SYMLINK` | Read path → `"b"`, follow |
| 2 | `b` | `T_SYMLINK` | Read path → `"a"`, follow |
| 3 | `a` | `T_SYMLINK` | Read path → `"b"`, follow |
| ... | ... | ... | ... |
| 10 | `b` | `T_SYMLINK` | Depth limit hit → return **`−1`** |

- `open()` must return **`−1`**.
- The kernel must **not** hang or panic.

**Responsible code:**
```c
int depth = 0;
while(ip->type == T_SYMLINK && !(omode & O_NOFOLLOW)){
  if(depth++ >= 10){       // ← guard triggers here on 11th attempt
    iunlockput(ip);
    end_op();
    return -1;
  }
  // ... readi, iunlockput, namei, ilock ...
}
```

---

#### Sub-task D — Symlink opened with `O_NOFOLLOW`

**Setup:**
```c
symlink("real_file", "mylink");
fd = open("mylink", O_RDONLY | O_NOFOLLOW);
```

**Required behaviour:**
- The `while` loop condition evaluates `!(omode & O_NOFOLLOW)` as **false**.
- The loop is **never entered**.
- `open()` returns a file descriptor pointing to the **symlink file itself**, not the target.
- Reading from this fd returns the raw path string (e.g., `"real_file"`), not the file's data.

> **Why this matters:** Tools like `ls -l` use `O_NOFOLLOW` to inspect the symlink itself and display what it points to, rather than following it.

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

> **Note:** The grade script only tests the circular and basic chain cases. The dangling symlink case and `O_NOFOLLOW` behaviour are **not** covered by the autograder — test them manually in `qemu` before submitting.

---

## Quick Reference

| Concept | Value |
|---|---|
| NDIRECT (after) | 11 |
| NINDIRECT | 256 |
| NDINDIRECT | 65536 (256×256) |
| MAXFILE | 65,803 |
| Doubly-indirect start block | 267 |
| `index1` formula | `bn / NINDIRECT` |
| `index2` formula | `bn % NINDIRECT` |
| Bitmap block coverage | 8192 blocks |
| `T_SYMLINK` | 4 |
| `O_NOFOLLOW` | `0x800` |
| Max symlink depth | 10 |
| `addrs[]` size | `NDIRECT + 2` = 13 |
| FSSIZE (after Task 1) | 200000 |
