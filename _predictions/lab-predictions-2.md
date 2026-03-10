---
layout: page
title: "Lab Predictions - P2R8Z"
lab: lab3
description: "Exam predictions for Lab3-w8: doubly-indirect blocks and symbolic links."
---

## Easy

<div class="var-question">
<div class="var-qnum">Q1 <span class="diff diff-easy">Easy</span></div>

What is the maximum file size in the **original** xv6 file system, and why is it limited to that value?

<button class="answer-toggle">▼ Show Answer</button>
<div class="answer-body">

The original maximum is **268 blocks** (~268 KB, since `BSIZE = 1024`).

This comes from the inode's address slots: 12 direct block pointers + 1 singly-indirect pointer. The singly-indirect block holds `BSIZE / sizeof(uint) = 1024/4 = 256` more addresses.

`12 + 256 = 268 blocks` total. Writing block 269 causes `bmap()` to return an error, which is why `bigfile` prints "file is too small" on unmodified xv6.

</div>
</div>

<div class="var-question">
<div class="var-qnum">Q2 <span class="diff diff-easy">Easy</span></div>

After the Lab3 modification, how many total blocks can a file have? Show the calculation.

<button class="answer-toggle">▼ Show Answer</button>
<div class="answer-body">

With `NDIRECT = 11`:

- 11 direct blocks
- 1 singly-indirect block → 256 data blocks
- 1 doubly-indirect block → 256 secondary map blocks × 256 data blocks each = 65,536 blocks

**Total: 11 + 256 + 65,536 = 65,803 blocks** (~64 MB)

We "pay" for the doubly-indirect by reducing direct blocks from 12 to 11 — sacrificing one direct slot to add 65,536 indirect slots.

</div>
</div>

<div class="var-question">
<div class="var-qnum">Q3 <span class="diff diff-easy">Easy</span></div>

What is a symbolic link and how does it differ from a regular file?

<button class="answer-toggle">▼ Show Answer</button>
<div class="answer-body">

A symbolic link (`T_SYMLINK`) is a special file whose **data blocks contain a path string** pointing to another file or directory. It holds no actual user data itself.

Differences from a regular file (`T_FILE`):
- A regular file stores data in its blocks. A symlink stores a **path string** in its blocks.
- Opening a regular file gives you its contents. Opening a symlink causes the kernel to **follow the path** and open the target instead.
- If the target is deleted, the symlink becomes a "dangling link" — opening it returns `-1` (target not found). A regular file's data persists until it is explicitly deleted.

</div>
</div>

<div class="var-question">
<div class="var-qnum">Q4 <span class="diff diff-easy">Easy</span></div>

What does `brelse()` do and why must it always be called after `bread()`?

<button class="answer-toggle">▼ Show Answer</button>
<div class="answer-body">

`brelse(bp)` releases a buffer back to the kernel's buffer cache (the "reading room"). The buffer cache has a fixed, small number of slots. Each `bread()` call locks a slot; if you never call `brelse()`, the cache fills up and the kernel will **panic** with "no buffers" on the next `bread()` call.

Rule: every `bread()` must be paired with exactly one `brelse()`, called as soon as you have extracted the address you need.

</div>
</div>

---

## Medium

<div class="var-question">
<div class="var-qnum">Q5 <span class="diff diff-med">Medium</span></div>

In `bmap()`, given logical block number `bn = 500` for the doubly-indirect section, calculate `index1` and `index2`. What do they represent?

<button class="answer-toggle">▼ Show Answer</button>
<div class="answer-body">

First, subtract the direct and singly-indirect ranges to get the local offset within the doubly-indirect region:

- Direct blocks: 11, singly-indirect: 256 → doubly-indirect starts at bn = 267.
- Local bn within doubly-indirect: `500 - 267 = 233`

Then:
- `index1 = 233 / 256 = 0` → use the **0th** secondary map block
- `index2 = 233 % 256 = 233` → use the **233rd** entry in that secondary map

`index1` selects which Secondary Map Book to open from the Master Map. `index2` selects which Data Block pointer inside that Secondary Map to return.

</div>
</div>

<div class="var-question">
<div class="var-qnum">Q6 <span class="diff diff-med">Medium</span></div>

Why must `struct inode` in `file.h` and `struct dinode` in `fs.h` always have the same number of elements in their `addrs[]` arrays?

<button class="answer-toggle">▼ Show Answer</button>
<div class="answer-body">

`struct dinode` is the **on-disk** format — it describes exactly how the inode is laid out in disk blocks. `struct inode` is the **in-memory** copy that the kernel works with.

When the kernel reads an inode from disk (`iget`/`ilock`), it copies the `addrs[]` array from `dinode` to `inode`. If the sizes differ, the copy will be **off by one or more entries**: some address slots won't be loaded into memory, or garbage values will be loaded into extra slots. This causes `bmap()` to return wrong block numbers — silent data corruption or a kernel panic.

After the lab modification, both must have `addrs[NDIRECT+2]` (11 direct + 1 indirect + 1 doubly-indirect = 13 slots).

</div>
</div>

<div class="var-question">
<div class="var-qnum">Q7 <span class="diff diff-med">Medium</span></div>

Explain the `O_NOFOLLOW` flag. When would a user program use it, and what happens without it when opening a symlink?

<button class="answer-toggle">▼ Show Answer</button>
<div class="answer-body">

`O_NOFOLLOW` tells `sys_open()` to open the **symlink file itself** rather than following the link to its target. Without this flag, `open()` on a symlink transparently redirects to the target.

Use cases for `O_NOFOLLOW`:
- Inspecting or deleting the link itself with `unlink()` — you need an fd to the link, not the target.
- Security auditing — verifying where a link points without being redirected.
- Tools like `ls -l` that display link metadata.

Without `O_NOFOLLOW` and without symlink-following code in `sys_open()`, opening `"link_to_a"` would just open the symlink's inode directly (type `T_SYMLINK`), which would then fail the `T_DIR` check or return wrong data — the lab's whole purpose is to add the following logic so this works correctly by default.

</div>
</div>

<div class="var-question">
<div class="var-qnum">Q8 <span class="diff diff-med">Medium</span></div>

Why must the symlink-following loop in `sys_open()` be placed **after** `ilock(ip)` but **before** the directory check?

<button class="answer-toggle">▼ Show Answer</button>
<div class="answer-body">

**After `ilock(ip)`**: You need the inode lock before reading `ip->type` or calling `readi()` on its data blocks. Without the lock, another process could be modifying the inode concurrently — reading `type` without the lock is a data race.

**Before the directory check** (`if(ip->type == T_DIR ...)`): A symlink can legally point to a **directory**. If you checked `T_DIR` first, a symlink pointing to a directory would incorrectly fail (it's not a directory itself — it's a `T_SYMLINK`). By resolving the link first and then checking the resulting inode's type, directory symlinks work correctly.

</div>
</div>

---

## Hard

<div class="var-question">
<div class="var-qnum">Q9 <span class="diff diff-hard">Hard</span></div>

Write the complete doubly-indirect block allocation logic for `bmap()`. Include the `balloc`, `bread`, `brelse`, and `log_write` calls.

<button class="answer-toggle">▼ Show Answer</button>
<div class="answer-body">

```c
// After singly-indirect handling:
bn -= NINDIRECT;
if (bn < NINDIRECT * NINDIRECT) {
    // Level 1: Master Map Block
    if ((addr = ip->addrs[NDIRECT+1]) == 0) {
        ip->addrs[NDIRECT+1] = addr = balloc(ip->dev);
        if (addr == 0) return 0;
    }
    struct buf *bp1 = bread(ip->dev, addr);
    uint *a1 = (uint*)bp1->data;

    // Calculate indices
    uint index1 = bn / NINDIRECT;   // which secondary map
    uint index2 = bn % NINDIRECT;   // which data block in that map

    // Level 2: Secondary Map Block
    if ((addr = a1[index1]) == 0) {
        a1[index1] = addr = balloc(ip->dev);
        if (addr == 0) { brelse(bp1); return 0; }
        log_write(bp1);  // mark master map as dirty
    }
    brelse(bp1);  // ALWAYS release before loading next block

    struct buf *bp2 = bread(ip->dev, addr);
    uint *a2 = (uint*)bp2->data;

    // Level 3: Data Block
    if ((addr = a2[index2]) == 0) {
        a2[index2] = addr = balloc(ip->dev);
        if (addr == 0) { brelse(bp2); return 0; }
        log_write(bp2);  // mark secondary map as dirty
    }
    brelse(bp2);
    return addr;
}
panic("bmap: out of range");
```

Key points: `brelse(bp1)` before `bread(bp2)` to avoid buffer cache exhaustion; `log_write` after every modification so changes survive a crash.

</div>
</div>

<div class="var-question">
<div class="var-qnum">Q10 <span class="diff diff-hard">Hard</span></div>

Write the doubly-indirect cleanup logic in `itrunc()`. What happens if you forget to free the Master Map block itself (only freeing the data blocks inside it)?

<button class="answer-toggle">▼ Show Answer</button>
<div class="answer-body">

```c
// In itrunc(), after singly-indirect cleanup:
if (ip->addrs[NDIRECT+1]) {
    struct buf *bp1 = bread(ip->dev, ip->addrs[NDIRECT+1]);
    uint *a1 = (uint*)bp1->data;
    for (int i = 0; i < NINDIRECT; i++) {
        if (a1[i]) {
            struct buf *bp2 = bread(ip->dev, a1[i]);
            uint *a2 = (uint*)bp2->data;
            for (int j = 0; j < NINDIRECT; j++) {
                if (a2[j]) bfree(ip->dev, a2[j]);  // free data blocks
            }
            brelse(bp2);
            bfree(ip->dev, a1[i]);  // free secondary map block
        }
    }
    brelse(bp1);
    bfree(ip->dev, ip->addrs[NDIRECT+1]);  // free master map block
    ip->addrs[NDIRECT+1] = 0;
}
```

If you forgot `bfree(ip->dev, ip->addrs[NDIRECT+1])`: the Master Map block (1 block) is leaked — never returned to the free list. Over time, creating and deleting many large files leaks one block each time. The free block list shrinks, and eventually `balloc()` finds no free blocks and **panics** (`"balloc: out of blocks"`). The leak is silent until the disk fills up.

</div>
</div>

<div class="var-question">
<div class="var-qnum">Q11 <span class="diff diff-hard">Hard</span></div>

In `sys_open()`, if a symlink chain is A → B → A (circular), what would happen without a depth limit? Implement the loop guard correctly.

<button class="answer-toggle">▼ Show Answer</button>
<div class="answer-body">

Without a depth limit, the kernel would follow A → B → A → B → A… indefinitely. Since this runs in the kernel with the inode lock held for each step, the kernel would **spin forever** (never sleeping, never making progress), consuming 100% CPU and making the system unresponsive. This is not a crash — it's an infinite loop in kernel space.

Correct implementation in `sys_open()`:

```c
#define MAX_SYMLINK_DEPTH 10
int depth = 0;
while (ip->type == T_SYMLINK && !(omode & O_NOFOLLOW)) {
    if (depth++ >= MAX_SYMLINK_DEPTH) {
        iunlockput(ip);
        return -1;  // "Too many levels of symbolic links"
    }
    // Read the target path from the symlink's data
    char target[MAXPATH];
    int n = readi(ip, 0, (uint64)target, 0, sizeof(target));
    iunlockput(ip);  // release old inode BEFORE looking up new one
    if (n <= 0) return -1;
    target[n] = 0;   // null-terminate

    // Follow the link
    if ((ip = namei(target)) == 0) return -1;
    ilock(ip);
}
```

The critical subtlety: `iunlockput(ip)` must be called **before** `namei(target)` — you cannot hold one inode lock while acquiring another, or you risk deadlock if two threads are traversing the same cycle in opposite directions.

</div>
</div>
