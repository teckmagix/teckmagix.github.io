---
layout: page
title: "Lab Predictions - lab3-w4j6"
lab: lab3
description: "Extended exam predictions for Lab3-w8: 12 new variations spanning easy MCQ-style, medium trace tasks, and hard implementation challenges across bmap, itrunc, symlink, open, and fs layout."
---

# ICT1012  
# OPERATING SYSTEMS  
# LABORATORY INSTRUCTIONS  
# Quiz — Lab3-w8 Predictions (Mega Pack)

---

## Contents

- [Variation 9 — True/False & MCQ Conceptual (Easy)](#variation-9)
- [Variation 10 — MAXFILE Arithmetic (Easy)](#variation-10)
- [Variation 11 — Buffer Cache: bread/brelse Sequence (Easy-Medium)](#variation-11)
- [Variation 12 — `bmap()` Boundary Block Numbers (Medium)](#variation-12)
- [Variation 13 — `itrunc()` bfree Count for Partial Files (Medium)](#variation-13)
- [Variation 14 — Symlink Registration Across All Files (Medium)](#variation-14)
- [Variation 15 — `sys_open()` Locking Order Trace (Medium-Hard)](#variation-15)
- [Variation 16 — `bmap()` Full Implementation from Scratch (Hard)](#variation-16)
- [Variation 17 — `itrunc()` Full Implementation from Scratch (Hard)](#variation-17)
- [Variation 18 — `sys_symlink()` Full Implementation from Scratch (Hard)](#variation-18)
- [Variation 19 — Combined Stress: Wrong NDIRECT Value Consequences (Hard)](#variation-19)
- [Variation 20 — Debug a Broken Implementation (Hard)](#variation-20)

---

<a name="variation-9"></a>
## Variation 9 — True/False & MCQ Conceptual (Easy, 7 marks)

### Prerequisite

No code changes required. This tests conceptual understanding of Lab3-w8.

---

### Context

Answer each question based on the xv6 file system after both Task 1 (large files) and Task 2 (symlinks) have been fully implemented.

---

### Task Requirements

#### True / False — answer T or F for each statement

**Q1.** After Task 1, `NDIRECT` is reduced from 12 to 11 because one direct slot is sacrificed to make room for the doubly-indirect pointer.

> **Answer: T**

---

**Q2.** `struct dinode` (on-disk) and `struct inode` (in-memory) can have different sizes for `addrs[]` as long as the kernel correctly converts between them.

> **Answer: F** — They must have exactly the same number of entries. `ilock()` copies raw bytes from disk into memory with no conversion. A mismatch silently reads wrong block addresses.

---

**Q3.** After Task 1, the singly-indirect slot moves from `addrs[12]` to `addrs[11]`.

> **Answer: T** — NDIRECT changes from 12 to 11, so the singly-indirect slot shifts from index 12 to index 11, and the new doubly-indirect slot occupies index 12.

---

**Q4.** A symbolic link file stores the target path in its inode's `type` field.

> **Answer: F** — The `type` field is set to `T_SYMLINK`. The target path string is stored in the **data blocks** of the inode, written using `writei()`.

---

**Q5.** `O_NOFOLLOW` causes `open()` to return `−1` when it encounters a symlink.

> **Answer: F** — `O_NOFOLLOW` causes `open()` to return a file descriptor pointing to **the symlink file itself**, not to return an error.

---

**Q6.** `brelse()` must be called on every buffer returned by `bread()`, even if no data was modified.

> **Answer: T** — `brelse()` returns the buffer to the buffer cache pool. If never released, the cache runs out of buffers and the kernel panics.

---

**Q7.** `balloc()` always returns a block filled with random data from the previous use of that disk block.

> **Answer: F** — `balloc()` returns a **zeroed** disk block. This is important for security (no data leakage between files) and correctness (zero entries in indirect blocks mean "not yet allocated").

---

#### Multiple Choice — circle the correct answer

**Q8.** Which function is responsible for translating a logical block number into a physical disk block address?

- A) `balloc()`  
- B) `bmap()`  
- C) `bread()`  
- D) `namei()`

> **Answer: B** — `bmap()` is the navigator. `balloc()` allocates new blocks, `bread()` reads a block into memory, `namei()` resolves a file path to an inode.

---

**Q9.** After Task 1, what is the maximum number of blocks a single file can use?

- A) 268  
- B) 65,536  
- C) 65,803  
- D) 66,061

> **Answer: C** — 11 direct + 256 singly-indirect + (256 × 256) doubly-indirect = 65,803.

---

**Q10.** What happens if the depth counter in `sys_open()` reaches 10 while following symlinks?

- A) The kernel panics  
- B) The kernel returns 0  
- C) The kernel returns −1 and stops following links  
- D) The kernel sleeps and retries

> **Answer: C** — The depth guard returns −1 to prevent infinite loops from circular symlinks.

---

### Task Testing

This is a written/conceptual variation — no code to run. Review your answers against the lab PDF and your implementation.

---

### Submission

Include your answers as comments in the submission document or verify they are reflected in your implementation choices.

---

<a name="variation-10"></a>
## Variation 10 — MAXFILE Arithmetic (Easy, 7 marks)

### Prerequisite

Same lab zip. The relevant file is `kernel/fs.h`.

---

### Context

1. `MAXFILE` defines the maximum number of blocks any single xv6 file can use.
2. After Task 1, the layout has three tiers: direct, singly-indirect, and doubly-indirect.
3. This task tests whether you can correctly calculate `MAXFILE` for different hypothetical inode layouts.

---

### Task Requirements

For each hypothetical layout below, calculate `MAXFILE`. Show your working.

---

#### Question 1 — Original xv6 (before Task 1)

```
NDIRECT   = 12
NINDIRECT = 256   (BSIZE=1024, sizeof(uint)=4 → 1024/4=256)
No doubly-indirect
```

```
MAXFILE = NDIRECT + NINDIRECT
        = 12 + 256
        = 268
```

---

#### Question 2 — After Task 1

```
NDIRECT     = 11
NINDIRECT   = 256
NDINDIRECT  = 256 × 256
```

```
MAXFILE = NDIRECT + NINDIRECT + NDINDIRECT
        = 11 + 256 + 65536
        = 65,803
```

---

#### Question 3 — Hypothetical: NDIRECT = 10, add doubly-indirect, BSIZE = 1024

```
NDIRECT     = 10
NINDIRECT   = 256
NDINDIRECT  = 256 × 256
```

```
MAXFILE = 10 + 256 + 65536 = 65,802
```

> Note: Reducing NDIRECT by 1 more costs 1 direct block but the doubly-indirect total stays the same. The tradeoff is 1 fewer direct block for the same indirect capacity.

---

#### Question 4 — Hypothetical: BSIZE = 512 (smaller block size)

```
NDIRECT     = 11
NINDIRECT   = 512 / 4 = 128
NDINDIRECT  = 128 × 128
```

```
MAXFILE = 11 + 128 + 16384 = 16,523
```

> Key insight: halving BSIZE quarters the doubly-indirect capacity (128² vs 256²), dramatically reducing MAXFILE.

---

#### Question 5 — Hypothetical: triply-indirect added, BSIZE = 1024

```
NDIRECT      = 11
NINDIRECT    = 256
NDINDIRECT   = 256 × 256
NTINDIRECT   = 256 × 256 × 256
```

```
MAXFILE = 11 + 256 + 65536 + 16,777,216 = 16,842,019 blocks ≈ 16 GB
```

> **Exam trap:** Students often calculate triply-indirect as 256³ but forget to add the direct and single/double tiers. Always sum all tiers.

---

#### Question 6 — What is MAXFILE if BSIZE = 2048?

```
NDIRECT     = 11
NINDIRECT   = 2048 / 4 = 512
NDINDIRECT  = 512 × 512
```

```
MAXFILE = 11 + 512 + 262144 = 262,667 blocks
```

---

### Task Testing

Verify your `MAXFILE` definition in `kernel/fs.h` after Task 1:

```c
#define NDINDIRECT  (NINDIRECT * NINDIRECT)
#define MAXFILE     (NDIRECT + NINDIRECT + NDINDIRECT)
```

Print it as a sanity check by temporarily adding to `kernel/main.c`:

```c
printf("MAXFILE = %d\n", MAXFILE);
```

Expected output during boot: `MAXFILE = 65803`

---

### Submission

Same as Variation 5.

---

<a name="variation-11"></a>
## Variation 11 — Buffer Cache: bread/brelse Sequence (Easy-Medium, 7 marks)

### Prerequisite

Same lab zip. The relevant file is `kernel/fs.c`. This task tests your understanding of the buffer cache rules, not implementation.

---

### Context

1. The xv6 buffer cache holds a **fixed number** of in-memory disk block buffers (defined by `NBUF` in `kernel/param.h`, default 30).
2. `bread(dev, blockno)` acquires a buffer, reads the block from disk, and **locks** the buffer — preventing other processes from using it until you release it.
3. `brelse(bp)` releases the lock and returns the buffer to the pool.
4. If all 30 buffers are locked simultaneously, the kernel panics with `bget: no buffers`.

---

### Task Requirements

For each code snippet, identify whether it is **correct** or **buggy**, and explain why.

---

#### Snippet 1

```c
struct buf *bp1 = bread(dev, block_a);
uint *a = (uint*)bp1->data;
uint addr = a[index];
brelse(bp1);
// use addr here
```

> **Verdict: CORRECT** — `bread()` loads the block, the address is read, `brelse()` releases the buffer immediately after. The buffer is free before `addr` is used.

---

#### Snippet 2

```c
struct buf *bp1 = bread(dev, block_a);
struct buf *bp2 = bread(dev, block_b);
uint *a = (uint*)bp1->data;
uint addr = a[index];
brelse(bp2);
brelse(bp1);
```

> **Verdict: CORRECT** — Both buffers are held simultaneously (which is allowed as long as NBUF is not exhausted), and both are released. However, releasing `bp2` before `bp1` is fine — `brelse` order does not matter.

---

#### Snippet 3

```c
struct buf *bp1 = bread(dev, block_a);
uint *a = (uint*)bp1->data;
struct buf *bp2 = bread(dev, block_b);
brelse(bp1);
// use bp2
brelse(bp2);
```

> **Verdict: CORRECT** — `bp1` is released before loading `bp2`. This is the pattern used in the doubly-indirect `bmap()` implementation: release the Master Map before loading the Secondary Map.

---

#### Snippet 4

```c
struct buf *bp1 = bread(dev, block_a);
uint *a = (uint*)bp1->data;
struct buf *bp2 = bread(dev, block_a);  // same block number!
brelse(bp1);
brelse(bp2);
```

> **Verdict: BUGGY — DEADLOCK** — `bread()` on the same block number while already holding it will deadlock. The buffer for `block_a` is already locked by `bp1`. `bread()` will spin forever waiting for the lock to be released, which can never happen since the same thread holds it.

---

#### Snippet 5

```c
struct buf *bp = bread(dev, block_a);
uint *a = (uint*)bp->data;
uint addr = a[5];
// forgot brelse(bp)
return addr;
```

> **Verdict: BUGGY — BUFFER LEAK** — The buffer is never released. After this function is called 30 times (NBUF=30), all buffers are permanently locked and the next `bread()` call will panic with `bget: no buffers`. This is the most common mistake in `bmap()` implementations.

---

#### Snippet 6 — The correct doubly-indirect sequence

```c
// Step 1: Load Master Map
struct buf *master = bread(dev, ip->addrs[NDIRECT+1]);
uint *a = (uint*)master->data;
uint secondary_addr = a[index1];

// Step 2: Release Master Map, load Secondary Map
brelse(master);
struct buf *secondary = bread(dev, secondary_addr);
uint *b = (uint*)secondary->data;
uint data_addr = b[index2];

// Step 3: Release Secondary Map
brelse(secondary);
return data_addr;
```

> **Verdict: CORRECT** — This is exactly the pattern required for doubly-indirect `bmap()`. Each buffer is released before the next one is loaded.

---

### Task Testing

If your `bmap()` implementation has a missing `brelse()`, the bug often only manifests after writing many blocks (the buffer cache gradually fills up). To catch it early, temporarily reduce `NBUF` in `kernel/param.h` to 5 and run `bigfile` — a buffer leak will panic much sooner.

---

### Submission

Same as Variation 5.

---

<a name="variation-12"></a>
## Variation 12 — `bmap()` Boundary Block Numbers (Medium, 7 marks)

### Prerequisite

Same lab zip. This task tests exact boundary conditions between the three address tiers.

---

### Context

After Task 1, the three tiers are:

| Tier | Block range (inclusive) | Count |
|---|---|---|
| Direct | 0 – 10 | 11 |
| Singly-indirect | 11 – 266 | 256 |
| Doubly-indirect | 267 – 65802 | 65536 |

The boundaries at blocks 10/11 and 266/267 are where off-by-one errors occur most frequently.

---

### Task Requirements

For each logical block number, state:
1. Which tier it falls in
2. The exact slot/index values used to locate it
3. Which `addrs[]` entry is the entry point

---

#### Block 0 (first direct block)

- Tier: **Direct**
- Entry point: `ip->addrs[0]`
- Index: none (direct lookup)

---

#### Block 10 (last direct block)

- Tier: **Direct**
- Entry point: `ip->addrs[10]`
- Index: none

---

#### Block 11 (first singly-indirect block)

- Tier: **Singly-indirect**
- Entry point: `ip->addrs[11]` (the singly-indirect map block)
- `bn_single = 11 - NDIRECT = 11 - 11 = 0`
- Index into map block: **0** (first slot)

---

#### Block 266 (last singly-indirect block)

- Tier: **Singly-indirect**
- Entry point: `ip->addrs[11]`
- `bn_single = 266 - 11 = 255`
- Index into map block: **255** (last slot)

---

#### Block 267 (first doubly-indirect block)

- Tier: **Doubly-indirect**
- Entry point: `ip->addrs[12]` (the Master Map block)
- `bn = 267 - 267 = 0`
- `index1 = 0 / 256 = 0` → Master Map slot 0
- `index2 = 0 % 256 = 0` → Secondary Map slot 0

---

#### Block 522 (last block in Secondary Map #0)

- Tier: **Doubly-indirect**
- `bn = 522 - 267 = 255`
- `index1 = 255 / 256 = 0` → Master Map slot 0 (still Secondary Map #0)
- `index2 = 255 % 256 = 255` → Secondary Map slot 255

---

#### Block 523 (first block in Secondary Map #1)

- Tier: **Doubly-indirect**
- `bn = 523 - 267 = 256`
- `index1 = 256 / 256 = 1` → Master Map slot 1 (Secondary Map #1)
- `index2 = 256 % 256 = 0` → Secondary Map slot 0

> **Exam trap:** Block 522 and block 523 sit on either side of a Secondary Map boundary. A student who computes `index1 = 255/256 = 0` for block 522 and `index1 = 256/256 = 1` for block 523 is correct — this is exactly where the Master Map transitions from pointing to Secondary Map #0 to Secondary Map #1.

---

#### Block 65802 (last block in the entire file system)

- Tier: **Doubly-indirect**
- `bn = 65802 - 267 = 65535`
- `index1 = 65535 / 256 = 255` → Master Map slot 255 (last Secondary Map)
- `index2 = 65535 % 256 = 255` → Secondary Map slot 255

---

#### Block 65803 (one beyond MAXFILE — should never be reached)

- Tier: **Beyond MAXFILE**
- `bmap()` should panic: `panic("bmap: out of range")`
- No valid index exists for this block number

---

### Task Testing

Add temporary `printf` statements to `bmap()` to trace which tier is entered for a given block number, then verify against the table above.

---

### Submission

Same as Variation 5.

---

<a name="variation-13"></a>
## Variation 13 — `itrunc()` bfree Count for Partial Files (Medium, 7 marks)

### Prerequisite

Same lab zip. The relevant file is `kernel/fs.c`.

---

### Context

1. `itrunc()` must free **exactly** the blocks that were allocated — no more, no less.
2. `itrunc()` checks each slot before calling `bfree()` — if a slot is `0`, that block was never allocated and must not be freed.
3. This task tests whether you can count the exact number of `bfree()` calls for various partial file sizes.

---

### Task Requirements

For each scenario, calculate the **exact number of `bfree()` calls** made by a correct `itrunc()`.

> Formula reminder:
> - Direct zone: 1 `bfree()` per non-zero `addrs[0..10]` entry
> - Singly-indirect zone: 1 `bfree()` per data block + 1 `bfree()` for the map block itself
> - Doubly-indirect zone: 1 `bfree()` per data block + 1 `bfree()` per Secondary Map + 1 `bfree()` for the Master Map

---

#### Scenario 1 — Empty file (0 blocks)

All `addrs[]` entries are 0. `itrunc()` checks each and skips all.

`bfree()` calls: **0**

---

#### Scenario 2 — 1 block file

Only `addrs[0]` is non-zero.

`bfree()` calls: **1** (just the one direct block)

---

#### Scenario 3 — Exactly 11 blocks (all direct, no indirect)

`addrs[0..10]` all non-zero. `addrs[11]` = 0.

`bfree()` calls: **11**

---

#### Scenario 4 — 12 blocks (all direct + 1 singly-indirect data block)

`addrs[0..10]` non-zero (11 blocks), `addrs[11]` non-zero (singly-indirect map), 1 entry in the map non-zero.

`bfree()` calls:
- 11 (direct data blocks)
- 1 (first singly-indirect data block)
- 1 (the singly-indirect map block itself)

**Total: 13**

---

#### Scenario 5 — 267 blocks (all direct + full singly-indirect, no doubly-indirect)

`addrs[0..10]` non-zero (11), `addrs[11]` non-zero with 256 entries filled.

`bfree()` calls:
- 11 (direct)
- 256 (singly-indirect data blocks)
- 1 (singly-indirect map block)

**Total: 268**

---

#### Scenario 6 — 268 blocks (all direct + full singly-indirect + 1 doubly-indirect data block)

`addrs[12]` non-zero (Master Map allocated), Master Map slot 0 non-zero (one Secondary Map allocated), Secondary Map slot 0 non-zero (one data block).

`bfree()` calls:
- 11 (direct)
- 257 (full singly-indirect: 256 data + 1 map)
- 1 (one doubly-indirect data block)
- 1 (Secondary Map #0)
- 1 (Master Map)

**Total: 271**

---

#### Scenario 7 — Full file (65,803 blocks)

All tiers fully allocated.

`bfree()` calls:
- 11 (direct)
- 257 (singly-indirect: 256 data + 1 map)
- 65,536 (doubly-indirect data blocks: 256 × 256)
- 256 (Secondary Map blocks)
- 1 (Master Map block)

**Total: 66,061**

---

#### Scenario 8 — 2 Secondary Maps fully used, rest empty

Direct: 11, singly-indirect: full (256+1=257), doubly-indirect: 2 Secondary Maps each with 256 data blocks.

`bfree()` calls:
- 11
- 257
- 2 × 256 = 512 (data blocks)
- 2 (Secondary Map blocks)
- 1 (Master Map block)

**Total: 783**

---

### Task Testing

To verify, add a counter variable to `itrunc()` and print it at the end, then cross-reference with the above scenarios by creating files of specific sizes.

---

### Submission

Same as Variation 5.

---

<a name="variation-14"></a>
## Variation 14 — Symlink Registration Across All Files (Medium, 7 marks)

### Prerequisite

Same lab zip. This variation focuses entirely on the **plumbing** required to register a new system call, without writing any kernel logic.

---

### Context

1. Every new system call in xv6 requires changes to **seven** different files before the kernel even recognises the call exists.
2. Missing any single step causes either a compile error, a link error, or a silent runtime failure (the call dispatches to the wrong handler).
3. This task tests whether you can identify every required change and explain why each one is necessary.

---

### Task Requirements

For each file below, state exactly what must be added and why it is necessary.

---

#### File 1 — `kernel/stat.h`

**What to add:**
```c
#define T_SYMLINK 4
```

**Why:** Defines the new file type constant. Without this, `create(path, T_SYMLINK, 0, 0)` cannot compile and the kernel has no way to distinguish symlinks from regular files when checking `ip->type`.

---

#### File 2 — `kernel/fcntl.h`

**What to add:**
```c
#define O_NOFOLLOW 0x800
```

**Why:** Defines the flag that tells `open()` to not follow a symlink. Must be a power-of-2 value with no bit overlap with existing flags (`O_WRONLY=0x1`, `O_RDWR=0x2`, `O_CREATE=0x200`, `O_TRUNC=0x400`). Value `0x800` is the next available bit.

---

#### File 3 — `kernel/syscall.h`

**What to add:**
```c
#define SYS_symlink 22
```

**Why:** Assigns a unique integer ID to the new system call. This number is what user space puts into a register before executing the `ecall` instruction. The kernel uses it to index into the syscall dispatch table. The number must be one higher than the current maximum (check existing definitions).

---

#### File 4 — `user/user.h`

**What to add:**
```c
int symlink(const char*, const char*);
```

**Why:** Declares the user-space function signature. Without this, any user program that calls `symlink()` will get a compile error: `implicit declaration of function 'symlink'`.

---

#### File 5 — `user/usys.pl`

**What to add:**
```perl
entry("symlink");
```

**Why:** This Perl script generates the assembly stub `user/usys.S`. The stub places `SYS_symlink` into register `a7` and executes `ecall` to transition from user mode to kernel mode. Without this entry, there is no compiled assembly bridge between the user-space `symlink()` call and the kernel handler.

---

#### File 6 — `kernel/syscall.c` (declaration)

**What to add:**
```c
extern uint64 sys_symlink(void);
```

**Why:** Forward-declares the kernel function. The `extern` keyword tells the compiler that `sys_symlink` is defined in another compilation unit (`kernel/sysfile.c`). Without this, the linker cannot find the function and the build fails.

---

#### File 7 — `kernel/syscall.c` (dispatch table)

**What to add:**
```c
[SYS_symlink] sys_symlink,
```

**Why:** Registers the handler in the dispatch table `syscalls[]`. When a user process calls `symlink()`, the kernel looks up `syscalls[SYS_symlink]` and calls it. Missing this entry means the syscall ID maps to a null pointer — the kernel will either crash or silently do nothing.

---

#### File 8 — `Makefile`

**What to add:**
```makefile
$U/_symlinktest\
```

**Why:** Adds `user/symlinktest.c` to the list of user programs compiled into the xv6 disk image. Without this line, `symlinktest` does not exist as a command inside xv6.

---

### Common Mistake Table

| Mistake | Symptom |
|---|---|
| Missing `stat.h` define | `T_SYMLINK` undeclared — compile error in `sysfile.c` |
| Wrong `O_NOFOLLOW` value | Flag overlaps existing flag — open() silently misbehaves |
| Missing `syscall.h` define | `SYS_symlink` undeclared — compile error |
| Missing `user.h` declaration | `symlinktest.c` fails to compile |
| Missing `usys.pl` entry | `symlink()` call from user space jumps to wrong syscall |
| Missing `extern` declaration | Linker error: undefined reference to `sys_symlink` |
| Missing dispatch table entry | `syscall()` calls null pointer — kernel panic |
| Missing `Makefile` entry | `symlinktest` not found in xv6 shell |

---

### Task Testing

```
$ make clean && make qemu
```

If all 8 changes are correct, the build will succeed with no errors and `symlinktest` will be available in the xv6 shell.

---

<a name="variation-15"></a>
## Variation 15 — `sys_open()` Locking Order Trace (Medium-Hard, 7 marks)

### Prerequisite

Same lab zip with symlink system call registered (Variation 14 complete). The relevant file is `kernel/sysfile.c`.

---

### Context

1. The xv6 kernel uses spin locks on inodes (`ilock`/`iunlock`) to prevent concurrent access.
2. Deadlocks occur when two threads each hold a lock the other needs, or when a single thread tries to acquire a lock it already holds.
3. The symlink traversal loop in `sys_open()` must acquire and release inode locks in a precise order.

---

### Task Requirements

Trace the lock state for each of the following scenarios through your `sys_open()` implementation.

---

#### Scenario 1 — Simple symlink: `open("link", O_RDONLY)` where link → real_file

| Step | Action | Lock state |
|---|---|---|
| 1 | `ip = namei("link")` | No lock held |
| 2 | `ilock(ip)` | `link` locked |
| 3 | Check `ip->type == T_SYMLINK` | True |
| 4 | `readi(ip, ...)` reads `"real_file"` | `link` still locked |
| 5 | `iunlock(ip)` | No lock held |
| 6 | `iput(ip)` | Reference dropped |
| 7 | `ip = namei("real_file")` | No lock held |
| 8 | `ilock(ip)` | `real_file` locked |
| 9 | Check `ip->type == T_SYMLINK` | False — exit loop |
| 10 | Continue with normal open | `real_file` locked |

> **Critical step: 5 before 7** — `iunlock` must happen before `namei`. If reversed, the link's lock is held while `namei` traverses the directory tree, which may try to acquire other locks and deadlock.

---

#### Scenario 2 — Self-referential symlink: `open("self", O_RDONLY)` where self → self

| Step | Action | Lock state |
|---|---|---|
| 1 | `ip = namei("self")` | No lock held |
| 2 | `ilock(ip)` | `self` locked |
| 3 | `readi` reads `"self"` | `self` still locked |
| 4 | `iunlock(ip)`, `iput(ip)` | No lock held |
| 5 | `ip = namei("self")` | No lock held (new reference) |
| 6 | `ilock(ip)` | `self` locked again (new acquisition) |
| 7 | depth++ → 1, 2, 3 ... 10 | depth reaches 10 |
| 8 | `iunlockput(ip)`, return −1 | No lock held |

> **Why no deadlock?** Because `iunlock + iput` happens **before** the next `namei`. Each iteration cleanly releases before reacquiring. The depth guard catches the infinite loop.

---

#### Scenario 3 — Buggy implementation (deadlock)

```c
// WRONG: lock held across namei
ilock(ip);
readi(ip, 0, (uint64)buf, 0, MAXPATH);
// forgot iunlock here
struct inode *next = namei(buf);   // ← namei may call ilock internally
ilock(next);                       // ← if next == ip, DEADLOCK
iunlock(ip);
```

Deadlock condition: `namei(buf)` resolves to the same inode as `ip`. `namei` calls `iget` which does not lock, but the subsequent `ilock(next)` tries to lock an inode that is already locked by the same thread → **spin forever**.

**Fix:**
```c
ilock(ip);
readi(ip, 0, (uint64)buf, 0, MAXPATH);
iunlock(ip);     // ← release BEFORE namei
iput(ip);
ip = namei(buf);
if(ip == 0) { end_op(); return -1; }
ilock(ip);       // ← now safe to lock new inode
```

---

#### Scenario 4 — What if `iunlock` is called but `iput` is forgotten?

```c
iunlock(ip);
// forgot iput(ip)
ip = namei(buf);
ilock(ip);
```

Result: **Reference count leak.** The inode's reference count (`ip->ref`) is never decremented. After following many links, inode reference counts are exhausted and the kernel panics with `iget: no inodes`. This is a slow memory leak — the bug may not manifest until many files are opened and closed.

---

### Task Testing

Test the locking implementation with the concurrent symlinks test:

```
$ symlinktest
Start: test concurrent symlinks
test concurrent symlinks: ok
```

The concurrent test creates multiple processes that simultaneously create and follow symlinks. Any locking bug will manifest as a panic or incorrect output here.

---

<a name="variation-16"></a>
## Variation 16 — `bmap()` Full Implementation from Scratch (Hard, 7 marks)

### Prerequisite

Same lab zip. Task: write the **complete** doubly-indirect section of `bmap()` from a blank slate with no hints.

---

### Context

You are given `bmap()` with the direct and singly-indirect sections intact but the entire doubly-indirect section removed and replaced with:

```c
// YOUR IMPLEMENTATION HERE
panic("bmap: doubly-indirect not implemented");
```

---

### Task Requirements

Write the complete doubly-indirect block allocation code. Your code must:

1. Handle the case where `addrs[NDIRECT+1]` is 0 (Master Map not yet allocated).
2. Load the Master Map with `bread()`.
3. Compute `index1` and `index2` correctly.
4. Handle the case where `Master Map[index1]` is 0 (Secondary Map not yet allocated) — allocate it and call `log_write()`.
5. Release the Master Map with `brelse()` **before** loading the Secondary Map.
6. Load the Secondary Map with `bread()`.
7. Handle the case where `Secondary Map[index2]` is 0 (data block not yet allocated) — allocate it and call `log_write()`.
8. Release the Secondary Map with `brelse()`.
9. Return the physical data block address.
10. Return `0` safely at each allocation failure point without leaking buffers.

---

### Reference Implementation

```c
// Doubly-indirect block
if(bn < NDINDIRECT){
  uint addr;
  struct buf *bp;
  uint *a;

  // Level 1: Master Map
  if((addr = ip->addrs[NDIRECT+1]) == 0){
    addr = balloc(ip->dev);
    if(addr == 0)
      return 0;
    ip->addrs[NDIRECT+1] = addr;
  }
  bp = bread(ip->dev, addr);
  a = (uint*)bp->data;

  uint index1 = bn / NINDIRECT;
  uint index2 = bn % NINDIRECT;

  // Level 2: Secondary Map
  if((addr = a[index1]) == 0){
    addr = balloc(ip->dev);
    if(addr == 0){
      brelse(bp);
      return 0;
    }
    a[index1] = addr;
    log_write(bp);
  }
  brelse(bp);

  bp = bread(ip->dev, addr);
  a = (uint*)bp->data;

  // Level 3: Data Block
  if((addr = a[index2]) == 0){
    addr = balloc(ip->dev);
    if(addr == 0){
      brelse(bp);
      return 0;
    }
    a[index2] = addr;
    log_write(bp);
  }
  brelse(bp);
  return addr;
}
panic("bmap: out of range");
```

---

### Common Mistakes in Full Implementations

| Mistake | Consequence |
|---|---|
| Using `NDIRECT` instead of `NDIRECT+1` for `addrs[]` index | Overwrites singly-indirect slot, corrupts all files |
| Forgetting `log_write(bp)` after updating a map entry | The map update is never written to disk — data loss on reboot |
| `brelse(bp)` after `log_write(bp)` — wrong order | `brelse` before `log_write` releases the buffer before marking it dirty — the write may never happen |
| Not checking `balloc()` return value | If disk is full, null block address 0 is stored silently |
| Returning before `brelse` on allocation failure | Buffer leak — 30 failures fill the buffer cache |

---

### Task Testing

```
$ ./grade-lab-fs bigfile
== Test running bigfile == running bigfile: OK (53.1s)
$
```

---

<a name="variation-17"></a>
## Variation 17 — `itrunc()` Full Implementation from Scratch (Hard, 7 marks)

### Prerequisite

Same lab zip. Task: write the complete doubly-indirect cleanup section of `itrunc()` from scratch.

---

### Context

You are given `itrunc()` with the direct and singly-indirect cleanup intact, but the doubly-indirect section removed:

```c
// YOUR IMPLEMENTATION HERE
// (doubly-indirect cleanup missing)
```

---

### Task Requirements

Write the complete doubly-indirect cleanup. Your code must:

1. Check `ip->addrs[NDIRECT+1]`. If zero, nothing to free — skip.
2. Load the Master Map with `bread()`.
3. Loop over all 256 Master Map entries:
   - If an entry is non-zero, load the Secondary Map.
   - Loop over all 256 Secondary Map entries and `bfree()` each non-zero data block.
   - `brelse()` the Secondary Map buffer.
   - `bfree()` the Secondary Map block.
4. `brelse()` the Master Map buffer.
5. `bfree()` the Master Map block.
6. Set `ip->addrs[NDIRECT+1] = 0`.

---

### Reference Implementation

```c
// Doubly-indirect blocks
if(ip->addrs[NDIRECT+1]){
  struct buf *master_bp = bread(ip->dev, ip->addrs[NDIRECT+1]);
  uint *master = (uint*)master_bp->data;

  for(int i = 0; i < NINDIRECT; i++){
    if(master[i]){
      struct buf *sec_bp = bread(ip->dev, master[i]);
      uint *sec = (uint*)sec_bp->data;

      for(int j = 0; j < NINDIRECT; j++){
        if(sec[j])
          bfree(ip->dev, sec[j]);
      }
      brelse(sec_bp);
      bfree(ip->dev, master[i]);
    }
  }
  brelse(master_bp);
  bfree(ip->dev, ip->addrs[NDIRECT+1]);
  ip->addrs[NDIRECT+1] = 0;
}
```

---

### Task Testing

```
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

Both runs must succeed. A broken `itrunc()` will cause the second run to fail due to exhausted disk blocks.

---

<a name="variation-18"></a>
## Variation 18 — `sys_symlink()` Full Implementation from Scratch (Hard, 7 marks)

### Prerequisite

Same lab zip with all 8 registration steps from Variation 14 complete.

---

### Task Requirements

Write the complete body of `sys_symlink(void)` in `kernel/sysfile.c` from scratch.

Your implementation must handle:
1. Fetching both string arguments from user space.
2. Beginning a file system transaction.
3. Creating the symlink inode.
4. Writing the target path into the inode's data blocks.
5. Releasing the inode.
6. Committing the transaction.
7. Returning 0 on success, −1 on any failure.

---

### Reference Implementation

```c
uint64
sys_symlink(void)
{
  char target[MAXPATH], path[MAXPATH];
  struct inode *ip;

  if(argstr(0, target, MAXPATH) < 0 || argstr(1, path, MAXPATH) < 0)
    return -1;

  begin_op();

  ip = create(path, T_SYMLINK, 0, 0);
  if(ip == 0){
    end_op();
    return -1;
  }

  if(writei(ip, 0, (uint64)target, 0, strlen(target)) != strlen(target)){
    iunlockput(ip);
    end_op();
    return -1;
  }

  iunlockput(ip);
  end_op();
  return 0;
}
```

---

### Error Handling Breakdown

| Failure point | Correct response |
|---|---|
| `argstr` fails (bad user pointer) | Return −1 immediately, no `begin_op()` yet |
| `create()` returns 0 (path conflict or disk full) | `end_op()` then return −1 |
| `writei()` writes fewer bytes than expected | `iunlockput(ip)`, `end_op()`, return −1 |
| Normal success | `iunlockput(ip)`, `end_op()`, return 0 |

> **Why `iunlockput` not `iunlock` + `iput` separately?** `iunlockput()` is a convenience function that does both. `create()` returns the inode **locked**, so you must unlock it. You also must `iput()` to decrement the reference count — otherwise the inode is never freed from memory.

---

### Task Testing

```
$ symlinktest
Start: test symlinks
test symlinks: ok
Start: test concurrent symlinks
test concurrent symlinks: ok
$
```

---

<a name="variation-19"></a>
## Variation 19 — Wrong NDIRECT Value Consequences (Hard, 7 marks)

### Prerequisite

Same lab zip. This is an analysis task — no correct code required. You must predict the consequences of specific incorrect changes.

---

### Context

This variation tests whether you understand **why** each change in Task 1 is required, by asking what breaks if specific changes are wrong or missing.

---

### Task Requirements

For each scenario, predict the exact failure mode.

---

#### Scenario A — `NDIRECT` changed to 11 in `kernel/fs.h` but `addrs[NDIRECT+1]` not updated to `addrs[NDIRECT+2]` in `struct dinode`

```c
// kernel/fs.h — WRONG
#define NDIRECT 11
// addrs still declared as:
uint addrs[NDIRECT+1];  // = addrs[12] — only 12 slots, not 13
```

**Consequence:**  
`addrs[12]` (the doubly-indirect slot) does not exist in the struct. Writing to `ip->addrs[NDIRECT+1]` (index 12) accesses **memory beyond the struct**, corrupting the next field in memory. This is undefined behaviour — likely corrupts the `size` field of adjacent inodes or causes a kernel panic on write.

---

#### Scenario B — `struct inode` updated to `addrs[NDIRECT+2]` but `struct dinode` not updated

```c
// kernel/fs.h — WRONG
struct dinode {
  ...
  uint addrs[NDIRECT+1];  // 12 entries on disk
};

// kernel/file.h — correct
struct inode {
  ...
  uint addrs[NDIRECT+2];  // 13 entries in memory
};
```

**Consequence:**  
`ilock()` copies `sizeof(struct dinode)` bytes from disk into the in-memory inode. The on-disk dinode has 12 `addrs[]` entries (48 bytes), but the in-memory struct expects 13 (52 bytes). The 13th entry (`addrs[12]`, the doubly-indirect pointer) is **never populated from disk** — it contains whatever garbage was in memory. Every doubly-indirect lookup either uses a random address or allocates a new block unnecessarily.

---

#### Scenario C — `FSSIZE` not updated from 2000 to 200000

**Consequence:**  
`bigfile` attempts to write 6580 blocks. The disk only has ~1954 data blocks. `balloc()` returns 0 (no free blocks) after the disk fills up. `bmap()` returns 0, `writei()` fails, and `bigfile` exits early with `write error` — the test fails. The `nmeta` line still shows `total 2000`.

---

#### Scenario D — `log_write()` omitted after allocating a new Secondary Map

```c
// WRONG — missing log_write
a[index1] = addr;
// log_write(bp);  ← omitted
brelse(bp);
```

**Consequence:**  
The Master Map update (writing the new Secondary Map block address into `a[index1]`) is **not recorded in the write-ahead log**. If the system crashes before the Master Map buffer is naturally flushed to disk, the entry remains 0 on disk. After reboot, `bmap()` thinks the Secondary Map was never allocated and allocates a **new** Secondary Map at a different address — the data blocks allocated under the old Secondary Map are **permanently leaked** (never freed, never accessible).

---

#### Scenario E — `brelse()` called before `log_write()`

```c
// WRONG — brelse before log_write
a[index1] = addr;
brelse(bp);      // ← released too early
log_write(bp);   // ← bp is no longer valid!
```

**Consequence:**  
After `brelse(bp)`, the buffer `bp` may be immediately reused by another `bread()` call in another process. The `bp` pointer now points to a different block's data. `log_write(bp)` marks the **wrong buffer** as dirty. The Master Map update may never reach disk, or another block's data is corrupted. This is a race condition — it may work sometimes and fail other times depending on scheduler timing.

---

### Task Testing

For each scenario, try introducing the bug deliberately, run `bigfile`, and observe the failure. This deepens your understanding beyond just making the tests pass.

---

<a name="variation-20"></a>
## Variation 20 — Debug a Broken Implementation (Hard, 7 marks)

### Prerequisite

Same lab zip. You are given a **pre-broken** implementation and must identify and fix the bugs.

---

### Context

A student submitted the following implementation of the doubly-indirect section of `bmap()` and `itrunc()`. The code compiles but fails tests. Find all bugs.

---

### Task Requirements

#### Buggy `bmap()` — find all bugs

```c
if(bn < NDINDIRECT){
  uint addr;
  struct buf *bp;
  uint *a;

  // BUG HUNT STARTS HERE
  if((addr = ip->addrs[NDIRECT]) == 0){          // Line A
    addr = balloc(ip->dev);
    ip->addrs[NDIRECT] = addr;                   // Line B
  }
  bp = bread(ip->dev, addr);
  a = (uint*)bp->data;

  uint index1 = bn / NINDIRECT;
  uint index2 = bn % NINDIRECT;

  if((addr = a[index1]) == 0){
    addr = balloc(ip->dev);
    a[index1] = addr;
    log_write(bp);
  }

  bp = bread(ip->dev, addr);                     // Line C
  a = (uint*)bp->data;
  brelse(bp);                                    // Line D (original brelse removed)

  if((addr = a[index2]) == 0){
    addr = balloc(ip->dev);
    a[index2] = addr;
    log_write(bp);
  }
  brelse(bp);
  return addr;
}
```

---

**Bug 1 — Lines A & B: `addrs[NDIRECT]` should be `addrs[NDIRECT+1]`**

`NDIRECT = 11`, so `addrs[NDIRECT]` is index 11 — the **singly-indirect slot**. The doubly-indirect pointer must be at index 12 (`NDIRECT+1`). This bug overwrites the singly-indirect pointer with the Master Map address, corrupting all singly-indirect block lookups.

---

**Bug 2 — Line C: `brelse(bp)` missing before `bp = bread(...)`**

The Master Map buffer (`bp`) is never released before loading the Secondary Map. Two buffers held simultaneously is allowed, but the Master Map buffer is **never released at all** — it leaks. After ~30 file writes, the buffer cache fills up.

**Fix:** Add `brelse(bp);` before Line C.

---

**Bug 3 — Line D: `brelse(bp)` called immediately after `bread()` for Secondary Map**

`brelse(secondary_bp)` is called before reading `a[index2]`. After `brelse`, the buffer data may be overwritten by another process. Reading `a[index2]` accesses freed memory — undefined behaviour.

**Fix:** Move `brelse(bp)` to after the Level 3 data block allocation, not immediately after `bread()`.

---

#### Buggy `itrunc()` — find all bugs

```c
if(ip->addrs[NDIRECT+1]){
  struct buf *master_bp = bread(ip->dev, ip->addrs[NDIRECT+1]);
  uint *master = (uint*)master_bp->data;

  for(int i = 0; i < NINDIRECT; i++){
    if(master[i]){
      struct buf *sec_bp = bread(ip->dev, master[i]);
      uint *sec = (uint*)sec_bp->data;

      for(int j = 0; j < NINDIRECT; j++){
        if(sec[j])
          bfree(ip->dev, sec[j]);
      }
      bfree(ip->dev, master[i]);               // Line E
      brelse(sec_bp);                          // Line F
    }
  }
  bfree(ip->dev, ip->addrs[NDIRECT+1]);        // Line G
  brelse(master_bp);                           // Line H
  ip->addrs[NDIRECT+1] = 0;
}
```

---

**Bug 4 — Lines E & F: `bfree` before `brelse`**

`bfree(ip->dev, master[i])` frees the disk block that `sec_bp` is currently reading from. Then `brelse(sec_bp)` releases a buffer that was backed by a now-freed block. The correct order is always: `brelse` first (release the in-memory buffer), then `bfree` (mark the disk block as free).

**Fix:** Swap Lines E and F: `brelse(sec_bp)` first, then `bfree(ip->dev, master[i])`.

---

**Bug 5 — Lines G & H: `bfree` before `brelse` for Master Map**

Same bug as Bug 4 but for the Master Map. `bfree` the disk block before `brelse`-ing the in-memory buffer.

**Fix:** Swap Lines G and H: `brelse(master_bp)` first, then `bfree(ip->dev, ip->addrs[NDIRECT+1])`.

---

### Summary of All Bugs Found

| # | Location | Bug | Fix |
|---|---|---|---|
| 1 | `bmap()` Lines A&B | `addrs[NDIRECT]` corrupts singly-indirect slot | Change to `addrs[NDIRECT+1]` |
| 2 | `bmap()` Line C | Master Map buffer never released | Add `brelse(bp)` before `bp = bread(...)` |
| 3 | `bmap()` Line D | Secondary Map released before reading index2 | Move `brelse` to after Level 3 allocation |
| 4 | `itrunc()` Lines E&F | `bfree` before `brelse` for Secondary Map | Swap to `brelse` then `bfree` |
| 5 | `itrunc()` Lines G&H | `bfree` before `brelse` for Master Map | Swap to `brelse` then `bfree` |

---

### Task Testing

```
$ make clean && make grade
Score: 100/100
$
```

---

## Quick Reference — All Variations

| Concept | Value |
|---|---|
| NDIRECT (after) | 11 |
| NINDIRECT | 256 |
| NDINDIRECT | 65,536 |
| MAXFILE | 65,803 |
| Direct block range | 0 – 10 |
| Singly-indirect range | 11 – 266 |
| Doubly-indirect range | 267 – 65,802 |
| `index1` = | `(bn - 267) / 256` or `bn / NINDIRECT` after adjusting bn |
| `index2` = | `(bn - 267) % 256` or `bn % NINDIRECT` after adjusting bn |
| Bitmap block coverage | 8,192 blocks per bitmap block |
| nmeta (FSSIZE=2000) | 46 |
| nmeta (FSSIZE=200000) | 70 |
| Bitmap blocks (FSSIZE=200000) | 25 |
| Full file bfree() calls | 66,061 |
| T_SYMLINK | 4 |
| O_NOFOLLOW | 0x800 |
| Max symlink depth | 10 |
| addrs[] array size | NDIRECT+2 = 13 |
| Files needing changes (symlink) | 8 files |
| Buffer cache size (NBUF) | 30 |
