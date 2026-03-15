---
layout: page
title: "Lab Predictions - lab3-x8y3"
lab: lab3
description: "Exam predictions for Lab3-w8: before vs after inode layout, symlink sys_open flow, grade test strings, and block leak analysis."
---

## Easy

<div class="var-question">
<div class="var-qnum">Q1 <span class="diff diff-easy">Easy</span></div>

The lab says the original bigfile output is <code>wrote 268 blocks / bigfile: file is too small</code>. Explain exactly why the kernel stops at 268 blocks — trace through what happens when bigfile tries to write block 269.

<div class="answer-content">

<pre><code class="language-c">// Original bmap() — no doubly-indirect:
static uint bmap(struct inode *ip, uint bn) {

    if (bn < NDIRECT) { /* direct: bn 0-11 */ return addr; }
    bn -= NDIRECT;                   // bn -= 12

    if (bn < NINDIRECT) { /* singly-indirect: bn 0-255 */ return addr; }

    // ← falls through here for bn >= 256 (i.e., logical block >= 268)
    panic("bmap: out of range");   // kernel panics on block 268+
}</code></pre>

<p><strong>What happens when bigfile writes block 269 (logical block 268):</strong></p>
<ol>
  <li><code>write(fd, buf, BSIZE)</code> → <code>writei(ip, ..., offset=268*1024)</code></li>
  <li><code>writei()</code> calls <code>bmap(ip, 268)</code></li>
  <li>In original <code>bmap()</code>: bn=268 → subtract NDIRECT (12) → bn=256 → 256 is NOT less than NINDIRECT (256) → falls through to <code>panic("bmap: out of range")</code></li>
  <li>Kernel panics</li>
</ol>

<p><strong>Why bigfile prints "wrote 268 blocks" not "wrote 269 blocks":</strong></p>
<p>The 268th call to <code>write()</code> succeeds (logical block 267 = singly-indirect slot 255, the last singly-indirect block). The 269th call triggers the panic. The test loop increments <code>blocks</code> only after a successful write, so the final count is 268.</p>

</div>
</div>

<div class="var-question">
<div class="var-qnum">Q2 <span class="diff diff-easy">Easy</span></div>

The lab describes the "Catalog Card" analogy: an inode is like a fixed-size card, and we cannot make it bigger. So instead we change how we use its 13 slots. Fill in this table showing the slot-by-slot mapping before and after the lab.

<div class="answer-content">

<table>
  <tr><th>Slot (addrs index)</th><th>Before lab (NDIRECT=12)</th><th>After lab (NDIRECT=11)</th></tr>
  <tr><td>0</td><td>Direct → Data Block 0</td><td>Direct → Data Block 0</td></tr>
  <tr><td>1–10</td><td>Direct → Data Blocks 1–10</td><td>Direct → Data Blocks 1–10</td></tr>
  <tr><td>11</td><td>Direct → Data Block 11 ← <em>sacrificed</em></td><td>Singly-indirect Map → 256 Data Blocks (11–266)</td></tr>
  <tr><td>12</td><td>Singly-indirect Map → 256 Data Blocks</td><td>Doubly-indirect Master Map → 256 Secondary Maps → 65,536 Data Blocks</td></tr>
</table>

<p><strong>The trade-off:</strong> Slot 11, which used to hold one direct data block (1 KB), is repurposed as the singly-indirect pointer. The old singly-indirect pointer (was at slot 12) is moved to make room for the new doubly-indirect pointer at slot 12. We lose 1 KB of direct storage but gain 64 MB of doubly-indirect capacity.</p>

<p><strong>Why the total slot count stays at 13:</strong> The struct size must not change. <code>struct dinode</code> is stored in fixed-size inode blocks on disk (16 inodes per block, each 64 bytes). Changing the array size would break the entire on-disk layout.</p>

</div>
</div>

<div class="var-question">
<div class="var-qnum">Q3 <span class="diff diff-easy">Easy</span></div>

What output does a fully correct submission produce when running <code>make clean && make grade</code>? Write the exact lines and what score is awarded.

<div class="answer-content">

<pre><code>$ make clean && make grade
== Test running bigfile ==
$ make qemu-gdb
running bigfile: OK (91.2s)
== Test running symlinktest ==
$ make qemu-gdb
(2.0s)
== Test symlinktest: symlinks ==
    symlinktest: symlinks: OK
== Test symlinktest: concurrent symlinks ==
    symlinktest: concurrent symlinks: OK
Score: 100/100</code></pre>

<p><strong>What the grader checks for each test:</strong></p>
<table>
  <tr><th>Test</th><th>String checked in output</th><th>Points</th></tr>
  <tr><td>running bigfile</td><td><code>"wrote 6580 blocks"</code> + <code>"bigfile done; ok"</code></td><td>40</td></tr>
  <tr><td>symlinktest: symlinks</td><td><code>"test symlinks: ok"</code></td><td>30</td></tr>
  <tr><td>symlinktest: concurrent symlinks</td><td><code>"test concurrent symlinks: ok"</code></td><td>30</td></tr>
</table>

<p><strong>Partial scores:</strong> If only Task 1 is implemented correctly: 40/100. If only Task 2: 60/100. Both tasks must pass for 100/100.</p>

</div>
</div>

<div class="var-question">
<div class="var-qnum">Q4 <span class="diff diff-easy">Easy</span></div>

In the symlink implementation, <code>create(path, T_SYMLINK, 0, 0)</code> is called. What do the four arguments mean and what does <code>create()</code> return?

<div class="answer-content">

<pre><code class="language-c">ip = create(path, T_SYMLINK, 0, 0);
//          ↑      ↑          ↑  ↑
//        path   type       major minor</code></pre>

<table>
  <tr><th>Argument</th><th>Meaning</th><th>Value here</th></tr>
  <tr><td><code>path</code></td><td>Where to create the new file (e.g. "/mylink")</td><td>The symlink's name passed to sys_symlink</td></tr>
  <tr><td><code>T_SYMLINK</code></td><td>The inode type to create</td><td>4 — marks this as a symbolic link</td></tr>
  <tr><td><code>0</code> (major)</td><td>Device major number — only used for T_DEVICE</td><td>0 — not a device</td></tr>
  <tr><td><code>0</code> (minor)</td><td>Device minor number — only used for T_DEVICE</td><td>0 — not a device</td></tr>
</table>

<p><strong>What create() returns:</strong></p>
<ul>
  <li>On success: a pointer to the new <code>struct inode</code> — already <strong>locked</strong> (ilock was called internally by create) with <code>ref = 1</code></li>
  <li>On failure (e.g. path already exists, disk full, no free inodes): returns 0</li>
</ul>

<p><strong>What create() does internally:</strong> calls <code>nameiparent()</code> to find the parent directory, verifies the name is free, calls <code>ialloc(T_SYMLINK)</code> to get a new inode, adds a directory entry via <code>dirlink()</code>, returns the locked inode.</p>

</div>
</div>

---

## Medium

<div class="var-question">
<div class="var-qnum">Q5 <span class="diff diff-med">Medium</span></div>

The <code>itrunc()</code> cleanup for the doubly-indirect section must call <code>bfree()</code> three types of blocks: data blocks, secondary map blocks, and the master map block. For a file with 1 full secondary map (256 data blocks) and no other doubly-indirect blocks, count exactly how many <code>bfree()</code> calls are made.

<div class="answer-content">

<pre><code class="language-c">// File state: master map block exists, only a1[0] non-zero (1 secondary map),
// all 256 slots in that secondary map are non-zero (256 data blocks)

if (ip->addrs[NDIRECT + 1]) {                  // master map exists
    bp1 = bread(ip->dev, ip->addrs[NDIRECT+1]);
    a1 = (uint*)bp1->data;

    for (i = 0; i < NINDIRECT; i++) {           // i = 0..255
        if (a1[i]) {                             // only i=0 is non-zero
            bp2 = bread(ip->dev, a1[0]);
            a2 = (uint*)bp2->data;

            for (j = 0; j < NINDIRECT; j++) {   // j = 0..255
                if (a2[j])                       // all 256 are non-zero
                    bfree(ip->dev, a2[j]);       // 256 calls  ← data blocks
            }
            brelse(bp2);
            bfree(ip->dev, a1[0]);               // 1 call  ← secondary map block
        }
        // i=1..255: a1[i] = 0, skipped
    }

    brelse(bp1);
    bfree(ip->dev, ip->addrs[NDIRECT+1]);        // 1 call  ← master map block
    ip->addrs[NDIRECT+1] = 0;
}</code></pre>

<table>
  <tr><th>What is freed</th><th>Count</th></tr>
  <tr><td>Data blocks (a2[0..255])</td><td>256</td></tr>
  <tr><td>Secondary map block (a1[0])</td><td>1</td></tr>
  <tr><td>Master map block</td><td>1</td></tr>
  <tr><td><strong>Total bfree() calls</strong></td><td><strong>258</strong></td></tr>
</table>

<p><strong>Note:</strong> <code>bfree()</code> itself calls <code>bread()</code>/<code>brelse()</code> internally to update the bitmap. Those are additional buffer cache operations, not counted above as separate <code>bfree()</code> calls.</p>

</div>
</div>

<div class="var-question">
<div class="var-qnum">Q6 <span class="diff diff-med">Medium</span></div>

In <code>sys_open()</code>, the symlink-following code calls <code>iunlockput(ip)</code> before <code>namei(target)</code>. What is <code>iunlockput()</code> and why must it happen before <code>namei()</code>, not after?

<div class="answer-content">

<p><strong>What iunlockput() does:</strong></p>
<pre><code class="language-c">void iunlockput(struct inode *ip) {
    iunlock(ip);    // Step 1: release the sleeplock
    iput(ip);       // Step 2: decrement ref count (may free inode if last ref and nlink==0)
}</code></pre>

<p><strong>Why BEFORE namei():</strong></p>
<p><code>namei(target)</code> walks the path tree, calling <code>ilock()</code> on each directory inode it visits. If we still hold the symlink's lock, this creates a potential <strong>lock ordering violation</strong>:</p>

<pre><code class="language-c">// WRONG — hold symlink lock while calling namei:
ilock(symlink_inode);          // holds lock A
// ... readi(target) ...
ip2 = namei(target);           // namei calls ilock on directory inodes
ilock(directory_inode);        // tries to acquire lock B

// Meanwhile, another process:
ilock(directory_inode);        // holds lock B
ip3 = namei("symlink");        // tries to acquire lock A
// → DEADLOCK: each process holds one lock and waits for the other</code></pre>

<p><strong>The correct sequence:</strong></p>
<pre><code class="language-c">readi(ip, 0, (uint64)target, 0, MAXPATH-1);  // read path (requires lock)
iunlockput(ip);   // ← release ALL locks before namei
ip = namei(target);   // now safe to acquire new locks
if (ip == 0) { end_op(); return -1; }
ilock(ip);        // lock the new inode for the next iteration</code></pre>

<p>xv6's general rule: never try to acquire lock B while holding lock A unless you can guarantee all processes acquire them in the same order. Releasing before namei eliminates the hold-and-wait condition entirely.</p>

</div>
</div>

<div class="var-question">
<div class="var-qnum">Q7 <span class="diff diff-med">Medium</span></div>

The lab shows this <code>ls</code> output after a successful bigfile run: <code>big.file 2 23 6737920</code>. What do each of the four fields mean, and how is 6737920 calculated?

<div class="answer-content">

<pre><code>big.file    2       23        6737920
  ↑         ↑        ↑           ↑
filename   type   inode#      size(bytes)</code></pre>

<table>
  <tr><th>Field</th><th>Value</th><th>Meaning</th></tr>
  <tr><td>filename</td><td>big.file</td><td>The name stored in the directory entry</td></tr>
  <tr><td>type</td><td>2</td><td><code>T_FILE = 2</code> — a regular data file</td></tr>
  <tr><td>inode#</td><td>23</td><td>The inode number allocated by <code>ialloc()</code> — the 23rd inode in the table (inodes 1–22 were used by the pre-built xv6 binaries)</td></tr>
  <tr><td>size</td><td>6737920</td><td>File size in bytes, stored in <code>ip->size</code></td></tr>
</table>

<p><strong>How 6737920 is calculated:</strong></p>
<pre><code class="language-c">TEST_BLOCKS = 65803 / 10 = 6580    // integer division

size = TEST_BLOCKS × BSIZE
     = 6580 × 1024
     = 6,737,920 bytes</code></pre>

<p><strong>Where size is updated:</strong> <code>writei()</code> increments <code>ip->size</code> by the number of bytes written in each call. After all 6580 blocks are written, <code>ip->size = 6737920</code>. <code>iupdate(ip)</code> writes this value back to the on-disk dinode.</p>

</div>
</div>

<div class="var-question">
<div class="var-qnum">Q8 <span class="diff diff-med">Medium</span></div>

The symlink loop uses <code>while (ip->type == T_SYMLINK && !(omode & O_NOFOLLOW))</code>. What are the two conditions that cause the loop to exit normally (not via the depth check)?

<div class="answer-content">

<p><strong>The two normal exit conditions:</strong></p>

<p><strong>Condition 1: <code>ip->type != T_SYMLINK</code></strong></p>
<p>The loop reaches a non-symlink inode. This happens when the chain ends at a regular file (<code>T_FILE</code>), directory (<code>T_DIR</code>), or device (<code>T_DEVICE</code>).</p>
<pre><code class="language-c">// Example: linkA → linkB → realfile
// After following linkB: ip = realfile's inode, ip->type = T_FILE
// Loop condition: T_FILE == T_SYMLINK? NO → exit loop
// sys_open() continues normally, allocates fd pointing to realfile</code></pre>

<p><strong>Condition 2: <code>omode & O_NOFOLLOW</code> is set</strong></p>
<p>The caller explicitly asked not to follow symlinks. Even if <code>ip->type == T_SYMLINK</code>, the loop never executes because the AND condition fails on the first check.</p>
<pre><code class="language-c">// Example: open("mylink", O_RDONLY | O_NOFOLLOW)
// ip = mylink's inode, ip->type = T_SYMLINK
// Loop condition: T_SYMLINK == T_SYMLINK → TRUE
//                BUT !(omode & O_NOFOLLOW) → !(1) → FALSE
// → FALSE AND TRUE = FALSE → loop never entered
// sys_open() opens the symlink inode itself (not the target)</code></pre>

<p><strong>The two abnormal exits:</strong></p>
<ul>
  <li><code>depth >= 10</code> → returns -1 (circular link error)</li>
  <li><code>namei(target) == 0</code> → returns -1 (dangling link error)</li>
</ul>

</div>
</div>

---

## Hard

<div class="var-question">
<div class="var-qnum">Q9 <span class="diff diff-hard">Hard</span></div>

Write the complete doubly-indirect section of <code>itrunc()</code>. A student writes it but puts <code>bfree(ip->dev, ip->addrs[NDIRECT+1])</code> BEFORE <code>brelse(bp1)</code>. What specific scenario causes corruption?

<div class="answer-content">

<p><strong>Correct code:</strong></p>
<pre><code class="language-c">if (ip->addrs[NDIRECT + 1]) {
    struct buf *bp1 = bread(ip->dev, ip->addrs[NDIRECT + 1]);
    uint *a1 = (uint *)bp1->data;

    for (int i = 0; i < NINDIRECT; i++) {
        if (a1[i]) {
            struct buf *bp2 = bread(ip->dev, a1[i]);
            uint *a2 = (uint *)bp2->data;
            for (int j = 0; j < NINDIRECT; j++) {
                if (a2[j]) bfree(ip->dev, a2[j]);
            }
            brelse(bp2);
            bfree(ip->dev, a1[i]);
        }
    }

    brelse(bp1);                                // ← CORRECT: release BEFORE bfree
    bfree(ip->dev, ip->addrs[NDIRECT + 1]);    // ← CORRECT: bfree AFTER brelse
    ip->addrs[NDIRECT + 1] = 0;
}</code></pre>

<p><strong>The buggy order (bfree before brelse) — corruption scenario:</strong></p>

<ol>
  <li>Process A: <code>bfree(dev, master_map_block_X)</code> — bitmap bit for block X is cleared to 0 (free)</li>
  <li>Process A still holds <code>bp1</code> (locked) pointing to block X's data in the buffer cache</li>
  <li>Process B: calls <code>balloc(dev)</code> → finds block X free in bitmap → allocates it → calls <code>bzero(dev, X)</code></li>
  <li><code>bzero()</code> calls <code>bread(dev, X)</code> → <code>bget()</code> finds the existing cache entry for block X (still held by Process A, <code>refcnt > 0</code>) → Process B <strong>sleeps</strong> waiting for <code>bp1</code> to be released</li>
  <li>Process A finishes the loop and calls <code>brelse(bp1)</code></li>
  <li>Process B wakes, zeroes block X, then writes its new file's data there</li>
  <li>Block X now belongs to Process B's new file</li>
  <li>But if Process A's loop index continued and tried to access more of <code>a1[]</code> after the bfree, it would be reading data from a block that now belongs to another file — the master map content has been overwritten by Process B's bzero</li>
</ol>

<p><strong>The rule:</strong> Release the buffer cache lock (<code>brelse</code>) before advertising the block as free (<code>bfree</code>). This ensures no one can reallocate the block while we still hold it.</p>

</div>
</div>

<div class="var-question">
<div class="var-qnum">Q10 <span class="diff diff-hard">Hard</span></div>

Suppose a student implements <code>sys_symlink()</code> but uses <code>argstr(1, target, MAXPATH)</code> and <code>argstr(0, path, MAXPATH)</code> — the arguments are swapped. What does this do to the file system, and what would a user see when they try to use the symlink?

<div class="answer-content">

<p><strong>The user-space call and argument ordering:</strong></p>
<pre><code class="language-c">// User calls: symlink("target_file", "link_name")
// In RISC-V calling convention:
//   a0 register = first argument  = "target_file"  (arg 0)
//   a1 register = second argument = "link_name"    (arg 1)

// argstr(0, ...) reads from a0 = "target_file"
// argstr(1, ...) reads from a1 = "link_name"</code></pre>

<p><strong>With swapped arguments:</strong></p>
<pre><code class="language-c">// Buggy code:
argstr(1, target, MAXPATH);  // target = "link_name"   ← WRONG
argstr(0, path,   MAXPATH);  // path   = "target_file" ← WRONG

// create(path, T_SYMLINK, 0, 0) creates a symlink AT "target_file"
// writei stores "link_name" in that symlink's data</code></pre>

<p><strong>What happens on disk:</strong></p>
<ul>
  <li>A new symlink file is created at the path <code>"target_file"</code> (instead of <code>"link_name"</code>)</li>
  <li>That symlink contains the path string <code>"link_name"</code> (instead of <code>"target_file"</code>)</li>
</ul>

<p><strong>What the user observes:</strong></p>
<pre><code class="language-c">// User code:
int fd = open("file_a", O_CREATE|O_RDWR); write(fd, "data", 4); close(fd);
symlink("file_a", "my_link");       // intended: my_link → file_a
// With bug: creates "file_a" symlink → pointing to "my_link" ← backwards!

// Now:
open("my_link", O_RDONLY);    // returns -1: my_link doesn't exist
open("file_a",  O_RDONLY);    // follows symlink file_a → looks for "my_link"
                               // "my_link" doesn't exist → returns -1

// ls output:
// file_a   T_SYMLINK   (the symlink was created here instead!)
// Original "file_a" regular file was DELETED and replaced by a symlink!</code></pre>

<p><strong>Worst case:</strong> If <code>"target_file"</code> already existed as a regular file, <code>create()</code> with the path pointing to it may either fail or truncate it — overwriting the user's data with a symlink inode.</p>

</div>
</div>

<div class="var-question">
<div class="var-qnum">Q11 <span class="diff diff-hard">Hard</span></div>

The lab says to allocate doubly-indirect blocks "only as needed, like the original bmap()". Explain what "only as needed" means in code, and what would happen to disk space if you eagerly allocated all 256 secondary maps whenever the first doubly-indirect block is needed.

<div class="answer-content">

<p><strong>What "only as needed" (lazy allocation) means:</strong></p>
<pre><code class="language-c">// CORRECT — lazy allocation:
// Only allocate secondary map when idx1 points to an empty slot
if ((addr = a[idx1]) == 0) {    // check: is this secondary map slot empty?
    addr = balloc(ip->dev);     // allocate ONLY this one secondary map
    if (addr) {
        a[idx1] = addr;
        log_write(bp);
    }
}</code></pre>

<p><strong>What eager allocation would look like (WRONG):</strong></p>
<pre><code class="language-c">// WRONG — eager allocation: pre-create all 256 secondary maps at once
bp_master = bread(ip->dev, master_map_addr);
a = (uint*)bp_master->data;
for (int k = 0; k < NINDIRECT; k++) {     // k = 0..255
    if (a[k] == 0) {
        a[k] = balloc(ip->dev);            // allocates 256 secondary maps!
        log_write(bp_master);
    }
}
brelse(bp_master);</code></pre>

<p><strong>Impact of eager allocation:</strong></p>
<p>Writing just the first doubly-indirect block (logical block 267) would allocate:</p>
<ul>
  <li>1 master map block</li>
  <li>256 secondary map blocks (all pre-allocated even though 255 are unused)</li>
  <li>1 data block for the actual write</li>
  <li>= <strong>258 disk blocks consumed for a 1-block write</strong></li>
</ul>

<p>With lazy allocation, the same write uses only 3 blocks (1 master map + 1 secondary map + 1 data block).</p>

<p><strong>Filesystem exhaustion calculation:</strong></p>
<pre><code>Data blocks available after FSSIZE=200000: 199,930

With eager allocation:
- First doubly-indirect write: 258 blocks used
- Can only do ~775 such writes before disk is full (199930 / 258 ≈ 775)
- But with only 65,803 file blocks maximum, most of those 256 secondary maps
  are WASTED for any file smaller than 65,536 doubly-indirect blocks (~64 MB)

With lazy allocation:
- Disk only fills in proportion to actual data written
- A 280-block file uses 1 master map + 1 secondary map + 13 data blocks = 15 extra blocks
- A 65803-block file uses 1 master map + 256 secondary maps + 65536 data blocks = correct</code></pre>

</div>
</div>
