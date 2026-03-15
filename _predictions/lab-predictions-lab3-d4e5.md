---
layout: page
title: "Lab Predictions - lab3-d4e5"
lab: lab3
description: "Exam predictions for Lab3-w8: step-by-step implementation, addrs mismatch bug, concurrent symlink test, and depth counter boundary."
---

## Easy

<div class="var-question">
<div class="var-qnum">Q1 <span class="diff diff-easy">Easy</span></div>

The lab says to update <code>kernel/file.h</code> to ensure <code>struct inode</code> has <code>addrs[NDIRECT+2]</code>. Why is this necessary, and what breaks if <code>struct inode</code> has a different array size from <code>struct dinode</code>?

<div class="answer-content">

<p><strong>Why they must match:</strong></p>
<p>When <code>ilock()</code> loads an inode from disk, it uses <code>memmove()</code> to copy the <code>addrs[]</code> array from the on-disk <code>dinode</code> into the in-memory <code>inode</code>:</p>

<pre><code class="language-c">// Inside ilock() in kernel/fs.c:
memmove(ip->addrs, dip->addrs, sizeof(ip->addrs));
//                              ↑ uses sizeof(struct inode's addrs)</code></pre>

<p><strong>If struct inode has addrs[NDIRECT+1] (old, 12 entries) while dinode has addrs[NDIRECT+2] (new, 13 entries):</strong></p>

<pre><code class="language-c">// sizeof(ip->addrs) = sizeof(uint[12]) = 48 bytes   ← wrong
// sizeof(dip->addrs) = sizeof(uint[13]) = 52 bytes

// memmove copies only 48 bytes
// ip->addrs[0..11] = copied correctly
// ip->addrs[12] = the doubly-indirect slot → NEVER COPIED!
// ip->addrs[12] stays 0 (zeroed stack memory)
// bmap() reads ip->addrs[NDIRECT+1] = 0 for every doubly-indirect block
// → balloc() called every time, allocating new master map blocks every write
// → all doubly-indirect data is lost between calls</code></pre>

<p><strong>Conversely, if dinode has 12 entries but inode has 13:</strong></p>
<p>The memmove copies 52 bytes but only 48 exist in the dinode's buffer region. The extra 4 bytes read garbage (adjacent struct fields or padding), and <code>ip->addrs[12]</code> gets a garbage value → <code>bmap()</code> tries to use it as a block number → kernel panic or filesystem corruption.</p>

<p><strong>Rule:</strong> Both arrays must always be <code>addrs[NDIRECT+2]</code> simultaneously.</p>

</div>
</div>

<div class="var-question">
<div class="var-qnum">Q2 <span class="diff diff-easy">Easy</span></div>

What does the <code>depth</code> counter in <code>sys_open()</code>'s symlink loop do, and what is the correct comparison: <code>depth >= 10</code> or <code>depth > 10</code>? How many symlinks can be followed successfully with each version?

<div class="answer-content">

<p><strong>The depth counter's purpose:</strong></p>
<p>Prevents infinite loops when symlinks form a cycle (A → B → A → ...). Without it, <code>sys_open()</code> would loop forever, exhausting the kernel stack.</p>

<p><strong>Comparison: depth >= 10 vs depth > 10</strong></p>

<pre><code class="language-c">// Version A: depth >= 10
int depth = 0;
while (ip->type == T_SYMLINK ...) {
    if (depth >= 10) { iunlockput(ip); end_op(); return -1; }
    depth++;
    // follow link...
}
// Iteration 1: depth=0, 0>=10? No, depth++ → depth=1, follow
// Iteration 2: depth=1, 1>=10? No, depth++ → depth=2, follow
// ...
// Iteration 10: depth=9, 9>=10? No, depth++ → depth=10, follow (10th link!)
// Iteration 11: depth=10, 10>=10? YES → return -1
// Maximum links followed: 10</code></pre>

<pre><code class="language-c">// Version B: depth > 10
while (ip->type == T_SYMLINK ...) {
    if (depth > 10) { return -1; }
    depth++;
    // follow link...
}
// Iteration 11: depth=10, 10>10? No, depth++ → depth=11, follow (11th link!)
// Iteration 12: depth=11, 11>10? YES → return -1
// Maximum links followed: 11</code></pre>

<table>
  <tr><th>Comparison</th><th>Max links followed</th><th>Error at iteration</th></tr>
  <tr><td><code>depth >= 10</code></td><td>10</td><td>11th</td></tr>
  <tr><td><code>depth > 10</code></td><td>11</td><td>12th</td></tr>
</table>

<p>The lab solution uses <code>depth >= 10</code>. Both are acceptable — the important thing is that the loop terminates. The symlinktest uses chains of at most 4 links, so either version passes the tests.</p>

</div>
</div>

<div class="var-question">
<div class="var-qnum">Q3 <span class="diff diff-easy">Easy</span></div>

The bigfile test prints one dot for every 100 blocks written. How many dots appear on a successful run, and why is <code>TEST_BLOCKS = 65803/10 = 6580</code> used instead of the full 65803?

<div class="answer-content">

<p><strong>Number of dots:</strong></p>
<pre><code class="language-c">#define TEST_BLOCKS (65803/10)   // = 6580

// Dots printed when blocks % 100 == 0
// blocks = 100, 200, 300, ..., 6500 → 65 dots
// blocks 6501-6580 → no dot (6580 % 100 = 80, not 0)
// Total dots: 65</code></pre>

<p>Successful output: 65 dots on one line, then <code>"wrote 6580 blocks"</code> then <code>"bigfile done; ok"</code>.</p>

<p><strong>Why 1/10th of the maximum:</strong></p>
<table>
  <tr><th>Reason</th><th>Details</th></tr>
  <tr><td>Time limit</td><td>Writing 65803 blocks can take 2–35 minutes depending on hardware. Gradescope has strict time limits per test.</td></tr>
  <tr><td>Still covers all code paths</td><td>6580 blocks requires 267+ doubly-indirect blocks, which fully exercises the new bmap() and itrunc() code. The full 65803 blocks would exercise the same code paths, just more times.</td></tr>
  <tr><td>Gradescope compatibility</td><td>The grader checks for the exact string <code>"wrote 6580 blocks"</code>. Using the old infinite loop or the full 65803 count would fail the grader string match.</td></tr>
</table>

<p><strong>What 6580 blocks covers:</strong></p>
<ul>
  <li>All 11 direct blocks (0–10) ✓</li>
  <li>All 256 singly-indirect blocks (11–266) ✓</li>
  <li>6313 doubly-indirect blocks (267–6579) — uses 25 secondary maps ✓</li>
</ul>

</div>
</div>

<div class="var-question">
<div class="var-qnum">Q4 <span class="diff diff-easy">Easy</span></div>

What is the <code>symlinktest</code> concurrent test checking, and how does it differ from the basic symlink test?

<div class="answer-content">

<p><strong>Basic test (test symlinks):</strong></p>
<ul>
  <li>Single process</li>
  <li>Creates a file, creates a symlink pointing to it, opens through the symlink</li>
  <li>Tests: basic link creation, chain links (4 levels deep), circular links (A→B→A), dangling links (target deleted)</li>
  <li>Checks correctness of the symlink follow logic</li>
</ul>

<p><strong>Concurrent test (test concurrent symlinks):</strong></p>
<ul>
  <li>Forks multiple child processes simultaneously</li>
  <li>Each child creates its own symlink and reads through it</li>
  <li>Tests the locking discipline: <code>ilock()</code>/<code>iunlock()</code> in <code>sys_open()</code> must prevent race conditions</li>
  <li>Checks that two processes following different symlinks simultaneously don't corrupt each other's inodes</li>
</ul>

<p><strong>What race condition it catches:</strong></p>
<p>Without proper locking, two processes could both be inside the symlink-following loop simultaneously. If Process A releases an inode's lock to follow a chain, and Process B allocates the same inode for a new file before A re-acquires it, A would follow a link into a file that now belongs to B — data corruption. The inode sleeplock prevents this.</p>

<p><strong>Grader output for passing both:</strong></p>
<pre><code class="language-c">$ ./grade-lab-fs symlink
== Test running symlinktest == (2.3s)
== Test symlinktest: symlinks ==
    symlinktest: symlinks: OK
== Test symlinktest: concurrent symlinks ==
    symlinktest: concurrent symlinks: OK</code></pre>

</div>
</div>

---

## Medium

<div class="var-question">
<div class="var-qnum">Q5 <span class="diff diff-med">Medium</span></div>

Trace through <code>itrunc()</code>'s doubly-indirect section for a file that has exactly 3 secondary maps, where secondary map 0 is full (256 data blocks), secondary map 1 has 100 data blocks, and secondary map 2 is empty. How many calls to <code>bfree()</code> are made in total?

<div class="answer-content">

<p><strong>Setup:</strong></p>
<pre><code class="language-c">// File uses logical blocks 267–622:
// Secondary map 0: a1[0] != 0, contains 256 data block pointers (all non-zero)
// Secondary map 1: a1[1] != 0, contains 100 data block pointers (slots 0-99 non-zero, 100-255 = 0)
// Secondary map 2: a1[2] = 0   (never allocated — not even a secondary map block exists)
// a1[3..255] = 0 (all empty)</code></pre>

<p><strong>Trace through the itrunc loop:</strong></p>
<pre><code class="language-c">bp1 = bread(dev, ip->addrs[NDIRECT+1]);   // load master map
a1 = (uint*)bp1->data;

for (i = 0; i < NINDIRECT; i++) {   // i = 0 to 255

    // i=0: a1[0] != 0 → enter inner loop
    bp2 = bread(dev, a1[0]);
    a2 = (uint*)bp2->data;
    for (j = 0; j < 256; j++) {
        if (a2[j]) bfree(dev, a2[j]);   // 256 calls to bfree (data blocks)
    }
    brelse(bp2);
    bfree(dev, a1[0]);                   // 1 call: free secondary map 0 block

    // i=1: a1[1] != 0 → enter inner loop
    bp2 = bread(dev, a1[1]);
    a2 = (uint*)bp2->data;
    for (j = 0; j < 256; j++) {
        if (a2[j]) bfree(dev, a2[j]);   // 100 calls to bfree (only slots 0-99 non-zero)
    }
    brelse(bp2);
    bfree(dev, a1[1]);                   // 1 call: free secondary map 1 block

    // i=2: a1[2] = 0 → SKIP (no bfree calls)
    // i=3..255: all 0 → SKIP
}

brelse(bp1);
bfree(dev, ip->addrs[NDIRECT+1]);       // 1 call: free master map block
ip->addrs[NDIRECT+1] = 0;</code></pre>

<p><strong>Total bfree() calls:</strong></p>
<table>
  <tr><th>Source</th><th>Count</th></tr>
  <tr><td>Data blocks in secondary map 0</td><td>256</td></tr>
  <tr><td>Data blocks in secondary map 1</td><td>100</td></tr>
  <tr><td>Secondary map 0 block itself</td><td>1</td></tr>
  <tr><td>Secondary map 1 block itself</td><td>1</td></tr>
  <tr><td>Master map block</td><td>1</td></tr>
  <tr><td><strong>Total</strong></td><td><strong>359</strong></td></tr>
</table>

</div>
</div>

<div class="var-question">
<div class="var-qnum">Q6 <span class="diff diff-med">Medium</span></div>

A student implements <code>sys_open()</code>'s symlink loop but calls <code>namei(target)</code> before <code>iunlockput(ip)</code>. Explain why this could cause a deadlock with the xv6 inode table.

<div class="answer-content">

<p><strong>The buggy code:</strong></p>
<pre><code class="language-c">// WRONG — namei called while ip is still locked:
while (ip->type == T_SYMLINK ...) {
    char target[MAXPATH];
    readi(ip, 0, (uint64)target, 0, MAXPATH-1);
    target[n] = '\0';

    struct inode *next = namei(target);  // ← WRONG: ip still locked!
    iunlockput(ip);                       // ← should be before namei
    ip = next;
    ilock(ip);
}</code></pre>

<p><strong>Why deadlock occurs:</strong></p>
<p><code>namei(target)</code> walks the path by calling <code>ilock()</code> on each directory inode it traverses. If the target path includes a directory whose inode is in the same inode table slot as <code>ip</code>, or if the symlink points back to itself:</p>

<ol>
  <li>Process A holds the sleeplock of inode #5 (the symlink)</li>
  <li>Process A calls <code>namei("file")</code></li>
  <li><code>namei</code> calls <code>dirlookup</code> on the root directory</li>
  <li>Root directory lookup calls <code>iget(ROOTDEV, ROOTINO)</code> → <code>ilock(root_inode)</code></li>
  <li>If the root inode is already locked by another process, Process A sleeps — while still holding inode #5's lock</li>
  <li>Process B, holding the root lock, tries to lock inode #5 → circular wait → <strong>deadlock</strong></li>
</ol>

<p><strong>The specific xv6 rule being violated:</strong></p>
<p>xv6 comments in <code>iget()</code>: "Caller must not hold a sleeplock when calling <code>ilock()</code> for the first time on a new inode". The lock ordering must be consistent — you must release the current inode before acquiring a new one in the same lock hierarchy.</p>

<p><strong>Correct order:</strong></p>
<pre><code class="language-c">iunlockput(ip);          // release FIRST
ip = namei(target);      // THEN resolve
if (ip == 0) { end_op(); return -1; }
ilock(ip);               // lock the new inode</code></pre>

</div>
</div>

<div class="var-question">
<div class="var-qnum">Q7 <span class="diff diff-med">Medium</span></div>

In <code>bmap()</code>, why must <code>brelse(bp)</code> be called on the master map buffer BEFORE calling <code>bread()</code> for the secondary map? What happens if you hold both buffers simultaneously?

<div class="answer-content">

<p><strong>The correct pattern:</strong></p>
<pre><code class="language-c">// Level 2: load master map, find secondary map address
bp = bread(ip->dev, master_map_addr);
a  = (uint*)bp->data;
if ((addr = a[idx1]) == 0) {
    addr = balloc(ip->dev);
    if (addr) { a[idx1] = addr; log_write(bp); }
}
brelse(bp);              // ← RELEASE master map here

if (addr == 0) return 0;

// Level 3: now safe to load secondary map
bp = bread(ip->dev, addr);
a  = (uint*)bp->data;
...</code></pre>

<p><strong>Why you cannot hold both simultaneously:</strong></p>
<p>The buffer cache has only <code>NBUF = 30</code> slots. Each <code>bread()</code> locks one slot. If every concurrent file operation holds 2 buffers at once:</p>

<pre><code class="language-c">// Worst case: 15 concurrent writes, each holding:
//   bp1 = master map (locked)
//   waiting for bp2 = secondary map

// 15 processes × 2 buffers = 30 buffer slots
// All 30 slots occupied by the 15 master maps
// Each process waiting for a secondary map slot → all sleeping
// No process can release its master map (it's waiting for secondary)
// → DEADLOCK: no progress possible
// → bget() panics: "bget: no buffers"</code></pre>

<p><strong>The general rule:</strong> In the xv6 buffer cache, acquire at most 1 buffer lock at a time. Release the current buffer before acquiring the next one. This prevents the classic "hold-and-wait" condition that leads to deadlock.</p>

<p><strong>Exception:</strong> <code>bpin()</code>/<code>bunpin()</code> don't hold the sleeplock — they just increment the reference count. These can overlap with a held buffer since they don't block.</p>

</div>
</div>

<div class="var-question">
<div class="var-qnum">Q8 <span class="diff diff-med">Medium</span></div>

Explain what <code>create(path, T_SYMLINK, 0, 0)</code> does inside <code>sys_symlink()</code>. What does it allocate, what does it lock, and what state is the returned inode in?

<div class="answer-content">

<p><strong>What create() does (from kernel/sysfile.c):</strong></p>
<pre><code class="language-c">// create(path, T_SYMLINK, 0, 0) does:
// 1. nameiparent(path, name)
//    → walks path to get parent directory inode + final component name
//    e.g. for "/mylink": parent = root dir "/", name = "mylink"

// 2. ilock(dp)  — lock the parent directory

// 3. dirlookup(dp, name, 0)
//    → check that "mylink" doesn't already exist in the directory
//    → if exists: iput + return the existing inode (or error for T_SYMLINK)

// 4. ialloc(dp->dev, T_SYMLINK)
//    → scan inode table for a free slot (type == 0)
//    → set type = T_SYMLINK, nlink = 1
//    → log_write the inode block
//    → iget() returns in-memory handle

// 5. ilock(ip)  — lock the new inode

// 6. dirlink(dp, name, ip->inum)
//    → adds ("mylink", inum) entry to the parent directory
//    → writei() on the directory data block

// 7. iunlockput(dp)  — release parent directory</code></pre>

<p><strong>State of the returned inode:</strong></p>
<table>
  <tr><th>Field</th><th>Value</th><th>Set by</th></tr>
  <tr><td><code>ip->type</code></td><td><code>T_SYMLINK</code></td><td><code>ialloc()</code></td></tr>
  <tr><td><code>ip->nlink</code></td><td>1</td><td><code>ialloc()</code></td></tr>
  <tr><td><code>ip->size</code></td><td>0</td><td>Initial (writei not called yet)</td></tr>
  <tr><td><code>ip->addrs[]</code></td><td>all 0</td><td>Zeroed by ialloc</td></tr>
  <tr><td>ip lock</td><td>HELD</td><td><code>ilock(ip)</code> inside create()</td></tr>
  <tr><td>ip->ref</td><td>1</td><td><code>iget()</code> inside ialloc</td></tr>
</table>

<p>After <code>create()</code> returns, the inode is locked and ready for <code>writei()</code> to store the target path string. <code>sys_symlink()</code> must call <code>iunlockput(ip)</code> when done.</p>

</div>
</div>

---

## Hard

<div class="var-question">
<div class="var-qnum">Q9 <span class="diff diff-hard">Hard</span></div>

The solution uses <code>NDINDIRECT = NINDIRECT * NINDIRECT = 256 * 256 = 65536</code>. If you compute <code>MAXFILE = NDIRECT + NINDIRECT + NDINDIRECT = 11 + 256 + 65536 = 65803</code>, verify that <code>bmap(ip, 65802)</code> (the very last block) navigates to the correct slot without going out of range.

<div class="answer-content">

<p><strong>Tracing bmap(ip, 65802) — the maximum logical block:</strong></p>
<pre><code class="language-c">// Starting bn = 65802

// Step 1: direct? 65802 < 11? NO
bn -= NDIRECT;       // bn = 65802 - 11 = 65791

// Step 2: singly-indirect? 65791 < 256? NO
bn -= NINDIRECT;     // bn = 65791 - 256 = 65535

// Step 3: doubly-indirect? 65535 < 65536? YES
idx1 = 65535 / 256 = 255   // integer division: 255.996... → 255
idx2 = 65535 % 256 = 255   // remainder: 255</code></pre>

<p><strong>Verification that indices are in range:</strong></p>
<table>
  <tr><th>Index</th><th>Value</th><th>Valid range</th><th>In range?</th></tr>
  <tr><td>idx1</td><td>255</td><td>0–255 (NINDIRECT-1)</td><td>✓ YES (last slot)</td></tr>
  <tr><td>idx2</td><td>255</td><td>0–255 (NINDIRECT-1)</td><td>✓ YES (last slot)</td></tr>
</table>

<p><strong>What bmap(ip, 65803) would do:</strong></p>
<pre><code class="language-c">bn = 65803
bn -= 11     → bn = 65792
bn -= 256    → bn = 65536
// doubly-indirect check: 65536 < 65536? NO
// Falls through to: panic("bmap: out of range");  ← correct!</code></pre>

<p><strong>The boundary makes sense:</strong></p>
<ul>
  <li>Block 65802 → idx1=255, idx2=255 → last entry of last secondary map → in range ✓</li>
  <li>Block 65803 → bn=65536 after subtractions → 65536 is NOT less than NDINDIRECT(65536) → panic ✓</li>
</ul>

<p>The doubly-indirect region covers exactly 65536 blocks (256 × 256), indexed 0–65535 after subtracting the direct and singly-indirect ranges. The MAXFILE constant correctly captures the total: 11 + 256 + 65536 = 65803 usable blocks (indices 0 to 65802).</p>

</div>
</div>

<div class="var-question">
<div class="var-qnum">Q10 <span class="diff diff-hard">Hard</span></div>

The Lab 3 solution modifies 5 files for Task 1 and 8 files for Task 2. If a student submits to Gradescope with Task 1 complete but Task 2 not started, what is the expected score and what exactly does the grader check?

<div class="answer-content">

<p><strong>Expected score: 40/100</strong></p>

<p><strong>What the grader checks:</strong></p>
<pre><code class="language-c">// From make grade output:
== Test running bigfile ==
    running bigfile: OK (91.2s)      ← 40 points if "wrote 6580 blocks" + "bigfile done; ok"

== Test running symlinktest ==
== Test symlinktest: symlinks ==
    symlinktest: symlinks: FAIL      ← 0 points (T_SYMLINK doesn't exist yet)
== Test symlinktest: concurrent symlinks ==
    symlinktest: concurrent symlinks: FAIL  ← 0 points

Score: 40/100</code></pre>

<p><strong>Why symlinktest fails without Task 2:</strong></p>
<p>Without <code>T_SYMLINK</code> defined in <code>stat.h</code>, the <code>symlinktest.c</code> program won't even compile. The Makefile won't build <code>$U/_symlinktest</code> without the type defined. The grader will see the test binary is missing and mark both symlink tests as FAIL.</p>

<p><strong>The grader's string checks:</strong></p>
<table>
  <tr><th>Test</th><th>String checked</th><th>Points</th></tr>
  <tr><td>bigfile</td><td><code>"wrote 6580 blocks"</code> AND <code>"bigfile done; ok"</code></td><td>40</td></tr>
  <tr><td>symlinktest: symlinks</td><td><code>"test symlinks: ok"</code></td><td>30</td></tr>
  <tr><td>symlinktest: concurrent symlinks</td><td><code>"test concurrent symlinks: ok"</code></td><td>30</td></tr>
</table>

<p><strong>Partial credit within Task 1:</strong></p>
<p>There is no partial credit — the bigfile test either passes entirely (writes exactly 6580 blocks) or fails. If the doubly-indirect implementation is missing, bigfile writes only 268 blocks (the singly-indirect limit), the grader sees <code>"wrote 268 blocks"</code> instead of <code>"wrote 6580 blocks"</code>, and awards 0 points for Task 1.</p>

</div>
</div>

<div class="var-question">
<div class="var-qnum">Q11 <span class="diff diff-hard">Hard</span></div>

A student's <code>bmap()</code> implementation allocates a new secondary map block even when the master map entry is non-zero (i.e., when the secondary map already exists). Write the incorrect code, trace through two consecutive writes to block 267 and show exactly what data corruption results.

<div class="answer-content">

<p><strong>The incorrect code:</strong></p>
<pre><code class="language-c">// WRONG — always allocates a new secondary map block:
if ((addr = a[idx1]) == 0) {
    addr = balloc(ip->dev);
    // ... log_write ...
}
// ↑ This is actually CORRECT (if == 0 check)

// The actual bug: student removes the == 0 check entirely:
addr = balloc(ip->dev);   // WRONG: always allocate new!
a[idx1] = addr;
log_write(bp);
brelse(bp);

bp = bread(ip->dev, addr);  // loads the newly allocated (zeroed) secondary map</code></pre>

<p><strong>Trace: First write to logical block 267 (doubly-indirect bn=0):</strong></p>
<pre><code class="language-c">// idx1 = 0, idx2 = 0
// a[0] is currently 0 (master map fresh)
// balloc() → secondary_map_A at block 500
// a[0] = 500, log_write(master_map)
// bread(500) → empty secondary map
// balloc() → data_block at block 501
// a2[0] = 501, log_write(secondary_map_A)
// Returns block 501 — write 267 → block 501
// State: master[0]=500, secondary_A[0]=501 ✓ (correct so far)</code></pre>

<p><strong>Trace: Second write to logical block 523 (doubly-indirect bn=256):</strong></p>
<pre><code class="language-c">// idx1 = 1, idx2 = 0
// a[1] is currently 0 (only secondary map 0 exists)
// Buggy code: balloc() ALWAYS called
// balloc() → secondary_map_B at block 502
// a[1] = 502 — OK for idx1=1, this is expected</code></pre>

<p><strong>Trace: Third write to logical block 267 again (second time):</strong></p>
<pre><code class="language-c">// idx1 = 0, idx2 = 0
// Buggy code: balloc() ALWAYS called (even though a[0]=500 already!)
// balloc() → NEW secondary_map_C at block 503
// a[0] = 503 (OVERWRITES 500!)
// log_write(master_map)
// bread(503) → empty (freshly allocated, zeroed)
// a2[0] = 0 → balloc() → new data block 504
// Returns block 504

// DATA CORRUPTION:
// Old write to block 267 was in block 501 (via secondary_map_A at block 500)
// New read from block 267 goes to block 504 (via secondary_map_C at block 503)
// The old data (in block 501) is now orphaned — unreachable but allocated
// Every read of logical block 267 now returns zeros (fresh block 504)</code></pre>

<p><strong>Summary of what went wrong:</strong> The lazy allocation check <code>if (addr == 0)</code> must remain. Without it, each access to an already-allocated level of the tree reallocates a new block, orphaning the old one and losing all previously written data. The <code>== 0</code> check is what makes bmap() idempotent for existing blocks.</p>

</div>
</div>
