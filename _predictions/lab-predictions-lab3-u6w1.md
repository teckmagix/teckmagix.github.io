---
layout: page
title: "Lab Predictions - lab3-u6w1"
lab: lab3
description: "Exam predictions for Lab3-w8: addrs[] slot layout, symlink creation flow, itrunc tree walk, and the write-ahead log."
---

## Easy

<div class="var-question">
<div class="var-qnum">Q1 <span class="diff diff-easy">Easy</span></div>

Draw the <code>addrs[]</code> array for a file with exactly 13 blocks. Show the slot index, what it stores, and the data it points to.

<div class="answer-content">

<pre><code class="language-c">// NDIRECT = 11, NINDIRECT = 256, addrs has 13 entries (indices 0..12)

addrs[0]   → Data Block for file byte  0 – 1023        (direct)
addrs[1]   → Data Block for file byte  1024 – 2047     (direct)
addrs[2]   → Data Block for file byte  2048 – 3071     (direct)
...
addrs[10]  → Data Block for file byte 10240 – 11263    (direct)  ← last direct slot
addrs[11]  → Singly-indirect Map Block                 (singly-indirect pointer)
               └── map[0] → Data Block for file byte 11264 – 12287  ← block 11
               └── map[1] → Data Block for file byte 12288 – 13311  ← block 12
               └── map[2..255] → 0  (only 2 singly-indirect blocks needed)
addrs[12]  → 0  (doubly-indirect slot unused — file is only 13 blocks)</code></pre>

<p><strong>Key points:</strong></p>
<ul>
  <li>A 13-block file uses 11 direct slots + the singly-indirect map block + 2 data blocks via singly-indirect = 14 disk blocks consumed in total (11 data + 1 map + 2 data)</li>
  <li>The doubly-indirect slot <code>addrs[12]</code> stays 0 — only allocated by <code>bmap()</code> when the first doubly-indirect block (logical block 267) is needed</li>
  <li>Lazy allocation: <code>bmap()</code> never touches a slot until a write actually needs that block</li>
</ul>

</div>
</div>

<div class="var-question">
<div class="var-qnum">Q2 <span class="diff diff-easy">Easy</span></div>

List the 5 files you must change for Task 1 (large files) and the single change made to each.

<div class="answer-content">

<table>
  <tr><th>File</th><th>Change</th></tr>
  <tr><td><code>kernel/param.h</code></td><td><code>FSSIZE</code>: 2000 → 200000</td></tr>
  <tr><td><code>kernel/fs.h</code></td><td>NDIRECT 12→11, add NDINDIRECT, update MAXFILE, change <code>dinode.addrs[NDIRECT+2]</code></td></tr>
  <tr><td><code>kernel/file.h</code></td><td>Change <code>inode.addrs[NDIRECT+2]</code> to match dinode</td></tr>
  <tr><td><code>kernel/fs.c</code> (bmap)</td><td>Add doubly-indirect block allocation after the singly-indirect section</td></tr>
  <tr><td><code>kernel/fs.c</code> (itrunc)</td><td>Add doubly-indirect tree walk to free all blocks on file deletion</td></tr>
</table>

<p><strong>Also required:</strong> <code>Makefile</code> must have <code>$U/_bigfile\</code> to compile the test binary (usually already present).</p>

<p><strong>Why both bmap and itrunc must change:</strong> <code>bmap()</code> handles allocation (growing a file); <code>itrunc()</code> handles deallocation (deleting a file). If only <code>bmap()</code> is updated, large files can be written but the disk will fill up after the first deletion because the doubly-indirect blocks are never freed.</p>

</div>
</div>

<div class="var-question">
<div class="var-qnum">Q3 <span class="diff diff-easy">Easy</span></div>

What does <code>balloc(dev)</code> return when the disk is full, and how does <code>bmap()</code> handle that return value?

<div class="answer-content">

<pre><code class="language-c">// balloc() when full:
static uint balloc(uint dev) {
    // ... scans bitmap, finds no free bit ...
    printf("balloc: out of blocks\n");
    return 0;   // ← returns 0 to signal failure
}</code></pre>

<p><strong>How bmap() handles return value 0:</strong></p>
<pre><code class="language-c">// In the doubly-indirect section of bmap():

// Level 1: allocate master map if needed
if ((addr = ip->addrs[NDIRECT + 1]) == 0) {
    addr = balloc(ip->dev);
    if (addr == 0) return 0;   // ← propagate failure up
    ip->addrs[NDIRECT + 1] = addr;
}

// Level 2: allocate secondary map if needed
if ((addr = a[idx1]) == 0) {
    addr = balloc(ip->dev);
    if (addr) {                // ← only update if allocation succeeded
        a[idx1] = addr;
        log_write(bp);
    }
}
brelse(bp);
if (addr == 0) return 0;     // ← check after brelse

// Level 3: allocate data block if needed
if ((addr = a[idx2]) == 0) {
    addr = balloc(ip->dev);
    if (addr) {
        a[idx2] = addr;
        log_write(bp);
    }
}
brelse(bp);
return addr;   // ← 0 if allocation failed, real block# if succeeded</code></pre>

<p><strong>When bmap returns 0:</strong> <code>writei()</code> sees the data block address is 0, stops writing, and returns fewer bytes than requested → <code>write()</code> returns a short count → the calling program (bigfile) detects failure.</p>

</div>
</div>

<div class="var-question">
<div class="var-qnum">Q4 <span class="diff diff-easy">Easy</span></div>

What does the <code>symlinktest</code> "Circle Test" check, and what output does it expect when the implementation is correct?

<div class="answer-content">

<p><strong>What it tests:</strong></p>
<p>Creates a circular symlink chain: A → B → A → B → ... When <code>open("A")</code> is called, the kernel follows the chain indefinitely unless there is a depth counter to stop it.</p>

<pre><code class="language-c">// In symlinktest.c (circle test):
symlink("B", "A");   // A points to B
symlink("A", "B");   // B points to A

int fd = open("A", 0);
if (fd >= 0) {
    // FAIL — should have returned -1
}
// PASS — open returned -1 (depth limit triggered)</code></pre>

<p><strong>Expected kernel behaviour:</strong> The symlink-following loop in <code>sys_open()</code> increments <code>depth</code> each iteration. When <code>depth >= 10</code>, it calls <code>iunlockput(ip)</code>, <code>end_op()</code>, and returns -1 to user space.</p>

<p><strong>Successful test output:</strong></p>
<pre><code>$ symlinktest
Start: test symlinks
test symlinks: ok
Start: test concurrent symlinks
test concurrent symlinks: ok</code></pre>

</div>
</div>

---

## Medium

<div class="var-question">
<div class="var-qnum">Q5 <span class="diff diff-med">Medium</span></div>

The lab PDF describes <code>bread()</code> as "opening a map book" and <code>brelse()</code> as "closing it". The Kernel's "Reading Room" has only 30 slots. Using this analogy, explain what happens when <code>bmap()</code> navigates three levels of the doubly-indirect tree.

<div class="answer-content">

<p><strong>Three "books" opened during navigation:</strong></p>
<ol>
  <li><strong>Level 1 — Open the Master Map Book:</strong> <code>bp1 = bread(dev, ip->addrs[12])</code>. You look inside to find which Secondary Map to use. Extract the address, then <strong>close it immediately</strong>: <code>brelse(bp1)</code>. You now hold 0 books.</li>
  <li><strong>Level 2 — Open the Secondary Map Book:</strong> <code>bp2 = bread(dev, secondary_map_addr)</code>. You look inside to find the data block address. Extract it, then <strong>close it</strong>: <code>brelse(bp2)</code>. You hold 0 books again.</li>
  <li><strong>Level 3 — The data block itself:</strong> Not opened by <code>bmap()</code> — <code>bmap()</code> just returns the physical block number. The caller (<code>writei</code>/<code>readi</code>) opens the data block separately.</li>
</ol>

<pre><code class="language-c">// Peak books held at any moment = 1
bp1 = bread(dev, master_map);      // open book 1 → Reading Room: 1 book in use
addr = ((uint*)bp1->data)[idx1];   // extract secondary map address
brelse(bp1);                       // close book 1 → Reading Room: 0 books in use

bp2 = bread(dev, addr);            // open book 2 → Reading Room: 1 book in use
data = ((uint*)bp2->data)[idx2];   // extract data block address
brelse(bp2);                       // close book 2 → Reading Room: 0 books in use

return data;</code></pre>

<p><strong>The critical rule:</strong> Never hold more than one "book" (buffer) open at a time. With 30 slots and potentially hundreds of concurrent operations, holding 2 books per operation would exhaust the Reading Room in just 15 concurrent writes.</p>

</div>
</div>

<div class="var-question">
<div class="var-qnum">Q6 <span class="diff diff-med">Medium</span></div>

The lab says to use <code>log_write(bp)</code> when updating map blocks in <code>bmap()</code>. What does <code>log_write()</code> actually do, and what would go wrong if you used <code>bwrite(bp)</code> instead?

<div class="answer-content">

<p><strong>What log_write() does:</strong></p>
<pre><code class="language-c">// log_write(bp) does NOT write to disk immediately. It:
// 1. Marks the buffer with B_DIRTY
// 2. Records its block number in the log header (log.lh.block[])
// 3. Pins the buffer in cache (prevents LRU eviction until committed)
// The actual disk write happens when end_op() triggers commit()</code></pre>

<p><strong>What bwrite() does:</strong></p>
<pre><code class="language-c">// bwrite(bp) calls virtio_disk_rw(bp, 1) immediately
// Writes the block directly to its real disk location
// Bypasses the transaction log entirely</code></pre>

<p><strong>What goes wrong with bwrite():</strong></p>
<p>Consider a write to logical block 267 that requires allocating a new master map, secondary map, and data block. Three separate disk writes are needed. If the machine crashes after the master map is written (bwrite #1) but before the secondary map is written (bwrite #2):</p>
<ul>
  <li>On reboot: master map exists on disk, pointing to a secondary map address that was never written</li>
  <li>The secondary map block's bit in the bitmap was set (by <code>balloc()</code>) but its content is zeroed (fresh allocation)</li>
  <li>Any read of logical block 267 follows the master map → reads the zeroed secondary map → finds address 0 → returns zeros instead of file data</li>
  <li>The file system is silently corrupted</li>
</ul>

<p><strong>Why log_write() prevents this:</strong> All three writes are staged in the log. They only reach their real disk locations after <code>end_op()</code> writes the commit record. A crash before the commit means all three writes are discarded on recovery — the file system returns to a consistent state.</p>

</div>
</div>

<div class="var-question">
<div class="var-qnum">Q7 <span class="diff diff-med">Medium</span></div>

Trace through <code>sys_open()</code>'s symlink loop when opening a 4-level chain: <code>d → c → b → a → file</code>. Show the value of <code>depth</code> and <code>ip</code> at the start of each iteration.

<div class="answer-content">

<pre><code class="language-c">// Setup: d→c→b→a→file (4 hops)
// Before loop:
ip = namei("d");     // d's inode, type=T_SYMLINK
ilock(ip);
depth = 0;</code></pre>

<table>
  <tr><th>Iteration</th><th>depth at start</th><th>ip (current)</th><th>target read</th><th>ip (after)</th></tr>
  <tr><td>1</td><td>0 (0 &lt; 10 → continue)</td><td>d (T_SYMLINK)</td><td>"c"</td><td>c (T_SYMLINK)</td></tr>
  <tr><td>2</td><td>1 (1 &lt; 10 → continue)</td><td>c (T_SYMLINK)</td><td>"b"</td><td>b (T_SYMLINK)</td></tr>
  <tr><td>3</td><td>2 (2 &lt; 10 → continue)</td><td>b (T_SYMLINK)</td><td>"a"</td><td>a (T_SYMLINK)</td></tr>
  <tr><td>4</td><td>3 (3 &lt; 10 → continue)</td><td>a (T_SYMLINK)</td><td>"file"</td><td>file (T_FILE)</td></tr>
  <tr><td>5 (check)</td><td>4</td><td>file (T_FILE)</td><td>—</td><td>Loop exits: T_FILE ≠ T_SYMLINK</td></tr>
</table>

<pre><code class="language-c">// After loop: ip = file's inode, ilock held, depth = 4
// Continues to directory check and filealloc()
// User receives a valid fd pointing to "file"</code></pre>

<p><strong>Key locking discipline each iteration:</strong></p>
<pre><code class="language-c">// Inside each iteration:
readi(ip, ...);          // read target path (requires ilock on current ip)
iunlockput(ip);          // release BEFORE namei — prevents deadlock
ip = namei(target);      // find next inode
ilock(ip);               // lock the new inode for next iteration's readi</code></pre>

</div>
</div>

<div class="var-question">
<div class="var-qnum">Q8 <span class="diff diff-med">Medium</span></div>

The lab hints say: "If the file system gets into a bad state, delete <code>fs.img</code> from Unix, not from xv6." Why can't you delete it from inside xv6?

<div class="answer-content">

<p><strong>What fs.img is:</strong> A file on your host (Linux/Windows) filesystem that xv6 uses as its virtual disk. QEMU mounts it as a block device and xv6 reads/writes it as if it were a real hard disk.</p>

<p><strong>Two different filesystems in play:</strong></p>
<table>
  <tr><th>Layer</th><th>Filesystem</th><th>Where fs.img lives</th></tr>
  <tr><td>Host OS (Linux)</td><td>ext4/NTFS/etc.</td><td>Your project directory: <code>c:/ICT1012/xv6labs-w8/fs.img</code></td></tr>
  <tr><td>xv6 (inside QEMU)</td><td>xv6 FS</td><td>The virtual disk that IS fs.img</td></tr>
</table>

<p><strong>Why <code>rm fs.img</code> from xv6 fails:</strong></p>
<pre><code class="language-c">// Inside xv6 shell:
$ rm fs.img
// xv6 looks for "fs.img" in its own filesystem (the virtual disk)
// fs.img is NOT a file stored inside the xv6 filesystem
// It IS the xv6 filesystem — the disk itself
// xv6 cannot see or reach host OS files

// rm fails: "no such file" or it tries to delete a file called
// "fs.img" inside xv6, which doesn't exist there</code></pre>

<p><strong>Correct procedure:</strong></p>
<pre><code class="language-c">// 1. Exit QEMU: Ctrl+A then X
// 2. From host Linux terminal:
$ rm fs.img      // deletes the file on the host filesystem
$ make qemu      // mkfs rebuilds fs.img from scratch with correct constants
                 // nmeta line will now reflect the updated NDIRECT/FSSIZE</code></pre>

</div>
</div>

---

## Hard

<div class="var-question">
<div class="var-qnum">Q9 <span class="diff diff-hard">Hard</span></div>

A student changes <code>NDIRECT</code> from 12 to 11 and recompiles the kernel but forgets to run <code>make clean</code>. The old <code>fs.img</code> (built with NDIRECT=12) is still present. Describe exactly what goes wrong when <code>ilock()</code> loads a file's inode from the stale disk.

<div class="answer-content">

<p><strong>The mismatch:</strong></p>
<table>
  <tr><th></th><th>Old fs.img (NDIRECT=12)</th><th>New kernel (NDIRECT=11)</th></tr>
  <tr><td>addrs[] meaning</td><td>[0..11]=direct, [12]=singly-indirect</td><td>[0..10]=direct, [11]=singly-indirect, [12]=doubly-indirect</td></tr>
  <tr><td>addrs[] size</td><td>13 entries (NDIRECT+1=13)</td><td>13 entries (NDIRECT+2=13)</td></tr>
</table>

<p><strong>What ilock() does with the stale inode:</strong></p>
<pre><code class="language-c">// ilock() copies disk dinode into in-memory inode:
memmove(ip->addrs, dip->addrs, sizeof(ip->addrs));
// sizeof(ip->addrs) = sizeof(uint[13]) = 52 bytes — same in both layouts
// The raw bytes are copied correctly

// But the kernel INTERPRETS them with the new layout:
// Old disk: addrs[11]=data_block_11, addrs[12]=singly_indirect_map
// New kernel reads: ip->addrs[11]=singly-indirect ptr, ip->addrs[12]=doubly-indirect ptr

// For a file that used the old singly-indirect:
// Old addrs[12] = address of the singly-indirect map block
// New kernel thinks ip->addrs[12] = address of doubly-indirect MASTER MAP
// New kernel thinks ip->addrs[11] = address of singly-indirect map

// What actually happened:
// The old singly-indirect map block is now interpreted as the master map
// bmap(ip, 267) reads ip->addrs[12] = old singly-indirect map address
// Treats that block as the master map → reads map[0] = data_block_11's address
// Uses data_block_11 as a secondary map → reads data_block_11's contents as block numbers
// Those block numbers are file data, not valid block addresses → follows random blocks
// → wrong data, panic on out-of-range block number, or silent corruption</code></pre>

<p><strong>Fix:</strong> Always <code>make clean</code> after changing <code>NDIRECT</code>. This deletes <code>fs.img</code>, forcing <code>mkfs</code> to rebuild it with the correct constants.</p>

</div>
</div>

<div class="var-question">
<div class="var-qnum">Q10 <span class="diff diff-hard">Hard</span></div>

The Gradescope grader runs <code>make grade</code> and checks for specific output strings. If a student submits with the original infinite-loop <code>bigfile.c</code> (no <code>#define TEST_BLOCKS</code>) but correct kernel changes, does the bigfile test pass or fail? Explain precisely.

<div class="answer-content">

<p><strong>The original bigfile.c (no TEST_BLOCKS):</strong></p>
<pre><code class="language-c">// Original — writes until write() fails:
while(1) {
    cc = write(fd, buf, sizeof(buf));
    if(cc <= 0) break;
    blocks++;
    if (blocks % 100 == 0) printf(".");
}
printf("\nwrote %d blocks\n", blocks);
if (blocks != 65803) {  // or similar check
    printf("bigfile: file is too small\n");
    exit(-1);
}</code></pre>

<p><strong>What happens with correct kernel but wrong bigfile.c:</strong></p>
<ol>
  <li>With doubly-indirect working, the kernel supports up to 65803 blocks</li>
  <li>The infinite loop runs until all 65803 blocks are written (~2–35 minutes)</li>
  <li>Prints: <code>"wrote 65803 blocks"</code> and <code>"bigfile done; ok"</code></li>
</ol>

<p><strong>What the Gradescope grader checks for:</strong></p>
<pre><code>== Test running bigfile ==
    running bigfile: OK   ← only if output contains "wrote 6580 blocks"</code></pre>

<p><strong>Result: FAIL</strong> — The grader checks for the exact string <code>"wrote 6580 blocks"</code> (the TEST_BLOCKS version). The student's output says <code>"wrote 65803 blocks"</code> — string mismatch → test marked as FAIL even though the kernel implementation is correct.</p>

<p><strong>Additionally:</strong> Writing 65803 blocks takes up to 35 minutes. Gradescope has a per-test time limit (typically ~3–5 minutes). The test would almost certainly time out before completing, resulting in a FAIL regardless of the output string.</p>

<p><strong>Lesson:</strong> The bigfile.c provided in the lab ZIP must be used as-is. Do not modify it back to the infinite-loop version.</p>

</div>
</div>

<div class="var-question">
<div class="var-qnum">Q11 <span class="diff diff-hard">Hard</span></div>

The <code>itrunc()</code> doubly-indirect section uses the pattern: <code>brelse(bp2)</code> then <code>bfree(ip->dev, a1[i])</code>. A student reverses it: <code>bfree()</code> first, then <code>brelse()</code>. Under what conditions does this cause data corruption, and why is the original order safe?

<div class="answer-content">

<p><strong>The buggy order:</strong></p>
<pre><code class="language-c">// WRONG order:
bfree(ip->dev, a1[i]);   // 1. mark secondary map block as FREE in bitmap
brelse(bp2);             // 2. release the buffer AFTER marking free</code></pre>

<p><strong>Race condition with a concurrent balloc():</strong></p>
<ol>
  <li>Process A (deleting file): calls <code>bfree(dev, secondary_map_block)</code></li>
  <li>The bitmap bit for <code>secondary_map_block</code> is now 0 (free)</li>
  <li>Process B (creating new file): calls <code>balloc(dev)</code>, finds <code>secondary_map_block</code> free → allocates it → calls <code>bzero(dev, secondary_map_block)</code></li>
  <li><code>bzero()</code> calls <code>bread(dev, secondary_map_block)</code> → gets the SAME buffer <code>bp2</code> that Process A still holds!</li>
  <li>But <code>bp2</code> is locked by Process A. Process B sleeps waiting for <code>brelse(bp2)</code></li>
  <li>Process A finally calls <code>brelse(bp2)</code> → Process B wakes, zeroes the buffer</li>
  <li>Process B then writes its own data to this block</li>
  <li>Result: the buffer now belongs to Process B's new file, but the itrunc loop continues using stale data from before the zero (if it somehow re-read before the zero) — or at minimum, Process B's new allocation is overwritten by subsequent itrunc cleanup</li>
</ol>

<p><strong>Why the correct order (brelse first, then bfree) is safe:</strong></p>
<pre><code class="language-c">// CORRECT order:
brelse(bp2);             // 1. release buffer — refcnt drops to 0, unlocked
bfree(ip->dev, a1[i]);  // 2. THEN mark as free

// When bfree() marks the block free:
// bp2 has refcnt=0 and is unlocked
// If balloc() immediately reallocates it and calls bzero():
// bzero() calls bread() → bget() finds the buffer in cache (valid=1)
// bget() REUSES the existing buffer slot (no need to read from disk)
// bzero() writes zeros to bp2->data
// brelse() — clean
// No race: Process A has already released bp2 before Process B can use it</code></pre>

<p><strong>The invariant:</strong> A buffer must be released from the cache before its block number is returned to the free pool. "Releasing" (brelse) means removing our claim on the buffer. "Freeing" (bfree) means advertising to the world that this block is available. The release must come first, or others can take the block while we still hold it.</p>

</div>
</div>
