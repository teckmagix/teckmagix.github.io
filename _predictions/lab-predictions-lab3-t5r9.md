---
layout: page
title: "Lab Predictions - lab3-t5r9"
lab: lab3
description: "Exam predictions for Lab3-w8: nmeta calculation, O_NOFOLLOW flags, bmap index tracing, itrunc block counting, and circular link deadlock."
---

## Easy

<div class="var-question">
<div class="var-qnum">Q1 <span class="diff diff-easy">Easy</span></div>

Before the lab, running <code>make clean && make qemu</code> prints <code>nmeta 46 ... total 2000</code>. After changing <code>FSSIZE</code> to 200000, what does the new nmeta line say and what changed?

<div class="answer-content">

<p><strong>Before (FSSIZE = 2000):</strong></p>
<pre><code>nmeta 46 (boot, super, log blocks 30 inode blocks 13, bitmap blocks 1) blocks 1954 total 2000</code></pre>

<p><strong>After (FSSIZE = 200000):</strong></p>
<pre><code>nmeta 70 (boot, super, log blocks 30 inode blocks 13, bitmap blocks 25) blocks 199930 total 200000</code></pre>

<p><strong>What changed and why:</strong></p>
<table>
  <tr><th>Component</th><th>Before</th><th>After</th><th>Why</th></tr>
  <tr><td>bitmap blocks</td><td>1</td><td>25</td><td>Each bitmap block tracks 8192 blocks (1024 bytes × 8 bits). 200000 ÷ 8192 = 24.4 → needs 25 blocks</td></tr>
  <tr><td>nmeta total</td><td>46</td><td>70</td><td>1+1+30+13+1 = 46 → 1+1+30+13+25 = 70</td></tr>
  <tr><td>data blocks</td><td>1954</td><td>199930</td><td>200000 − 70 = 199930</td></tr>
  <tr><td>log, inode blocks</td><td>unchanged</td><td>unchanged</td><td>Fixed by LOGSIZE and NINODES constants</td></tr>
</table>

</div>
</div>

<div class="var-question">
<div class="var-qnum">Q2 <span class="diff diff-easy">Easy</span></div>

What do these four constants represent after the lab changes? Give the numeric value of each.

<pre><code class="language-c">#define NDIRECT    ___
#define NINDIRECT  ___
#define NDINDIRECT ___
#define MAXFILE    ___</code></pre>

<div class="answer-content">

<pre><code class="language-c">#define NDIRECT    11                        // direct block slots in addrs[] (was 12)
#define NINDIRECT  (BSIZE / sizeof(uint))    // = 1024 / 4 = 256
#define NDINDIRECT (NINDIRECT * NINDIRECT)   // = 256 × 256 = 65,536
#define MAXFILE    (NDIRECT + NINDIRECT + NDINDIRECT)  // = 11 + 256 + 65,536 = 65,803</code></pre>

<p><strong>What each represents physically:</strong></p>
<ul>
  <li><strong>NDIRECT = 11</strong> — the first 11 slots in <code>addrs[]</code> point directly to data blocks</li>
  <li><strong>NINDIRECT = 256</strong> — one map block holds 256 block-number entries (each is a 4-byte uint, and a block is 1024 bytes)</li>
  <li><strong>NDINDIRECT = 65536</strong> — the doubly-indirect region: 256 secondary maps × 256 data blocks each</li>
  <li><strong>MAXFILE = 65803</strong> — the absolute maximum blocks any file can have after the lab</li>
</ul>

</div>
</div>

<div class="var-question">
<div class="var-qnum">Q3 <span class="diff diff-easy">Easy</span></div>

The lab adds <code>T_SYMLINK = 4</code> to <code>kernel/stat.h</code>. What are all four file type values in xv6 after the lab, and where is a file's type physically stored?

<div class="answer-content">

<pre><code class="language-c">// kernel/stat.h (after lab)
#define T_DIR     1   // Directory
#define T_FILE    2   // Regular file
#define T_DEVICE  3   // Device file (e.g. /dev/console)
#define T_SYMLINK 4   // Symbolic link  ← new in Lab 3</code></pre>

<p><strong>Where the type is stored:</strong></p>
<p>In the <code>type</code> field of <code>struct dinode</code> on disk and <code>struct inode</code> in RAM. Both structs start with:</p>
<pre><code class="language-c">struct dinode {
    short type;   // ← T_SYMLINK = 4 stored here on disk
    // ...
};</code></pre>

<p><strong>How it's read:</strong> When <code>ilock(ip)</code> runs, it calls <code>bread()</code> to load the inode block from disk and copies <code>dip->type</code> into <code>ip->type</code>. The kernel then checks <code>ip->type == T_SYMLINK</code> inside <code>sys_open()</code> to decide whether to follow the link.</p>

<p><strong>Special value:</strong> type = 0 means the inode slot is <em>free</em>. <code>ialloc()</code> scans for type == 0 when allocating a new inode.</p>

</div>
</div>

<div class="var-question">
<div class="var-qnum">Q4 <span class="diff diff-easy">Easy</span></div>

The bigfile test prints 65 dots on success. What does each dot represent, and what is the formula to get 65?

<div class="answer-content">

<pre><code class="language-c">// Inside bigfile.c:
#define TEST_BLOCKS (65803/10)  // = 6580

blocks = 0;
for (i = 0; i < TEST_BLOCKS; i++) {
    write(fd, buf, BSIZE);
    blocks++;
    if (blocks % 100 == 0)
        printf(".");            // one dot every 100 blocks
}</code></pre>

<p><strong>Each dot = 100 blocks × 1024 bytes = 100 KB successfully written.</strong></p>

<p><strong>Formula for 65 dots:</strong></p>
<pre><code>TEST_BLOCKS / 100 = 6580 / 100 = 65.8 → 65 complete hundreds
(the remaining 80 blocks after the 65th dot don't trigger another dot)</code></pre>

<p><strong>Why 1/10th of 65803:</strong> Writing all 65803 blocks takes 2–35 minutes depending on hardware. Gradescope has a time limit, so the test is capped at 6580 blocks (~6 MB instead of ~64 MB). All three addressing levels (direct, singly-indirect, doubly-indirect) are still exercised.</p>

</div>
</div>

---

## Medium

<div class="var-question">
<div class="var-qnum">Q5 <span class="diff diff-med">Medium</span></div>

Trace <code>bmap(ip, 267)</code> step by step — the very first doubly-indirect block. Show all subtractions and what value <code>bn</code> has at each stage.

<div class="answer-content">

<pre><code class="language-c">// Initial value
bn = 267

// Step 1: Is it direct? bn < NDIRECT (11)? → 267 < 11? NO
bn -= NDIRECT;   // 267 − 11 = 256

// Step 2: Is it singly-indirect? bn < NINDIRECT (256)? → 256 < 256? NO
bn -= NINDIRECT;  // 256 − 256 = 0

// Step 3: Is it doubly-indirect? bn < NDINDIRECT (65536)? → 0 < 65536? YES
idx1 = 0 / 256 = 0   // → Secondary Map #0
idx2 = 0 % 256 = 0   // → slot #0 in that map</code></pre>

<p><strong>Navigation path:</strong></p>
<ol>
  <li><code>ip->addrs[12]</code> → address of Master Map block (allocate if 0)</li>
  <li><code>bread(master_map)</code> → load master map; read <code>a[idx1=0]</code></li>
  <li><code>brelse(master_map)</code> before loading next level</li>
  <li><code>bread(secondary_map_0)</code> → load secondary map #0; read <code>a[idx2=0]</code></li>
  <li><code>brelse(secondary_map_0)</code></li>
  <li>Return the data block address</li>
</ol>

<p><strong>Why logical block 267 is the first doubly-indirect:</strong> Blocks 0–10 are direct (11 blocks). Blocks 11–266 are singly-indirect (256 blocks). Block 267 = 11+256 = first block beyond both → enters the doubly-indirect region with bn=0 after both subtractions → idx1=0, idx2=0 → first slot of first secondary map.</p>

</div>
</div>

<div class="var-question">
<div class="var-qnum">Q6 <span class="diff diff-med">Medium</span></div>

A student runs bigfile, it succeeds (6580 blocks). They delete <code>big.file</code> and run bigfile again. The second run only writes 268 blocks. Their <code>bmap()</code> is correct. What is wrong?

<div class="answer-content">

<p><strong>Root cause: <code>itrunc()</code> doubly-indirect section is missing or incomplete.</strong></p>

<p>When bigfile runs a second time, it opens <code>big.file</code> with <code>O_CREATE</code>. Since the file already exists, the kernel calls <code>itrunc(ip)</code> to reset it to size 0 before writing. If <code>itrunc()</code> doesn't free the doubly-indirect blocks:</p>

<ol>
  <li>Direct blocks (0–10): freed correctly → 11 blocks returned to free pool</li>
  <li>Singly-indirect blocks (11–266): freed correctly → 1 map + 256 data = 257 blocks returned</li>
  <li>Doubly-indirect blocks (267–6579): <strong>NOT freed</strong> → still marked as "used" in the bitmap</li>
</ol>

<p>When the second bigfile run tries to write block 267 (first doubly-indirect), <code>bmap()</code> calls <code>balloc()</code> to allocate a master map. But the bitmap shows all those blocks as still in use → <code>balloc()</code> can't find free blocks for the doubly-indirect region. The file system appears full after 268 blocks (the freed direct + singly-indirect region).</p>

<p><strong>Fix: add the doubly-indirect tree walk to <code>itrunc()</code>:</strong></p>
<pre><code class="language-c">if (ip->addrs[NDIRECT + 1]) {
    // bread master map, loop through secondary maps,
    // bfree each data block, bfree each secondary map,
    // brelse master map, bfree master map,
    // ip->addrs[NDIRECT + 1] = 0
}</code></pre>

</div>
</div>

<div class="var-question">
<div class="var-qnum">Q7 <span class="diff diff-med">Medium</span></div>

The lab says <code>O_NOFOLLOW</code> must not overlap with existing flags. Given these existing values, choose a valid value for <code>O_NOFOLLOW</code> and explain why each existing value is disqualified.

<pre><code class="language-c">#define O_RDONLY  0x000
#define O_WRONLY  0x001
#define O_RDWR    0x002
#define O_CREATE  0x200
#define O_TRUNC   0x400</code></pre>

<div class="answer-content">

<p><strong>Correct answer: <code>O_NOFOLLOW = 0x800</code></strong></p>

<p><strong>Why each existing value is disqualified:</strong></p>
<table>
  <tr><th>Value</th><th>Binary</th><th>Why disqualified</th></tr>
  <tr><td>0x001</td><td>0000 0001</td><td>Taken by O_WRONLY — setting O_NOFOLLOW would silently make every open write-only</td></tr>
  <tr><td>0x002</td><td>0000 0010</td><td>Taken by O_RDWR</td></tr>
  <tr><td>0x200</td><td>0010 0000 0000</td><td>Taken by O_CREATE</td></tr>
  <tr><td>0x400</td><td>0100 0000 0000</td><td>Taken by O_TRUNC</td></tr>
</table>

<p><strong>Why 0x800 works:</strong></p>
<pre><code class="language-c">0x800 = 1000 0000 0000   // bit 11 — no existing flag uses this bit

// Test in sys_open():
if (omode & O_NOFOLLOW)  // only true when bit 11 is set
// Cannot be triggered by any combination of O_RDONLY/WRONLY/RDWR/CREATE/TRUNC
// since none of those have bit 11 set</code></pre>

<p>Flags are combined with <code>|</code> (bitwise OR), so any flag that shares a bit with another becomes indistinguishable. Each flag must occupy a unique bit.</p>

</div>
</div>

<div class="var-question">
<div class="var-qnum">Q8 <span class="diff diff-med">Medium</span></div>

In <code>sys_symlink()</code>, after <code>writei()</code> stores the target path, the code does NOT write a null terminator to disk. Yet <code>sys_open()</code> works correctly. Explain why.

<div class="answer-content">

<p><strong>What sys_symlink() stores:</strong></p>
<pre><code class="language-c">// Writes exactly strlen(target) bytes — no '\0'
writei(ip, 0, (uint64)target, 0, strlen(target));
// e.g. for target = "file": stores 'f','i','l','e' (4 bytes)</code></pre>

<p><strong>What sys_open() does after reading:</strong></p>
<pre><code class="language-c">char target[MAXPATH];
int n = readi(ip, 0, (uint64)target, 0, MAXPATH - 1);
// n = 4 (bytes read = bytes written = strlen of original path)
target[n] = '\0';   // ← kernel adds the null terminator here!</code></pre>

<p><strong>Why this is correct:</strong></p>
<ol>
  <li><code>readi()</code> returns the exact number of bytes stored (<code>n = strlen(original_target)</code>)</li>
  <li>The code immediately does <code>target[n] = '\0'</code>, placing the terminator at the right position</li>
  <li><code>namei(target)</code> then receives a properly null-terminated string</li>
</ol>

<p><strong>What breaks if you forget <code>target[n] = '\0'</code>:</strong></p>
<pre><code class="language-c">// Without null termination:
char target[MAXPATH];   // stack memory — contains garbage past position n
readi(ip, 0, (uint64)target, 0, MAXPATH - 1);
// target[0..n-1] = "file"
// target[n..MAXPATH-2] = whatever was on the stack

ip = namei(target);   // reads until '\0' — may read "file" + garbage
// namei fails to find "file\x??garbage..." → returns 0 → open returns -1</code></pre>

</div>
</div>

---

## Hard

<div class="var-question">
<div class="var-qnum">Q9 <span class="diff diff-hard">Hard</span></div>

For a file that uses exactly logical blocks 0 through 522 (523 blocks total), calculate how many calls to <code>bfree()</code> <code>itrunc()</code> makes in the doubly-indirect section. Show the breakdown.

<div class="answer-content">

<p><strong>First, classify the 523 blocks:</strong></p>
<pre><code>Blocks 0–10:    direct (11 blocks)
Blocks 11–266:  singly-indirect (256 blocks)
Blocks 267–522: doubly-indirect (256 blocks)</code></pre>

<p><strong>Doubly-indirect breakdown (blocks 267–522 = 256 blocks):</strong></p>
<pre><code class="language-c">// After subtracting NDIRECT + NINDIRECT: bn range = 0..255
// idx1 = bn / 256: for bn=0..255, idx1 = 0 for ALL of them
// → only Secondary Map #0 is used
// idx2 = bn % 256: for bn=0..255, idx2 = 0..255 (all 256 slots)</code></pre>

<p><strong>What itrunc() does in the doubly-indirect section:</strong></p>
<table>
  <tr><th>Action</th><th>Count</th></tr>
  <tr><td>bfree() on data blocks (a2[0..255])</td><td>256</td></tr>
  <tr><td>bfree() on Secondary Map #0 block</td><td>1</td></tr>
  <tr><td>bfree() on Master Map block</td><td>1</td></tr>
  <tr><td><strong>Total bfree() calls in doubly-indirect section</strong></td><td><strong>258</strong></td></tr>
</table>

<p><strong>Note:</strong> The outer loop runs from <code>i=0</code> to <code>i=255</code> (checking all 256 master map entries). Only <code>i=0</code> has a non-zero entry (<code>a1[0] != 0</code>), so entries <code>a1[1..255]</code> are all zero and their inner loops are never entered. The complete itrunc for this file: 258 (doubly-indirect) + 257 (singly-indirect: 1 map + 256 data) + 11 (direct) = <strong>526 total bfree() calls</strong>.</p>

</div>
</div>

<div class="var-question">
<div class="var-qnum">Q10 <span class="diff diff-hard">Hard</span></div>

A student's <code>bmap()</code> forgets to call <code>brelse(bp1)</code> before loading the secondary map. Explain exactly what happens after 15 concurrent processes each call <code>write(fd, buf, BSIZE)</code> on a new doubly-indirect block.

<div class="answer-content">

<p><strong>The bug:</strong></p>
<pre><code class="language-c">bp1 = bread(ip->dev, master_map);   // locks buffer slot
a   = (uint*)bp1->data;
// ... brelse(bp1) is MISSING ...
bp2 = bread(ip->dev, secondary_map);  // tries to lock another slot</code></pre>

<p><strong>What happens per process:</strong> Each process holds bp1 (master map) while waiting for a free slot for bp2 (secondary map). That uses 2 buffer slots per process instead of 1.</p>

<p><strong>Progression to panic (NBUF = 30 slots):</strong></p>
<table>
  <tr><th>Processes doing doubly-indirect write</th><th>Buffer slots used</th><th>Free slots</th></tr>
  <tr><td>1</td><td>2</td><td>28</td></tr>
  <tr><td>10</td><td>20</td><td>10</td></tr>
  <tr><td>14</td><td>28</td><td>2</td></tr>
  <tr><td>15</td><td>30</td><td>0</td></tr>
  <tr><td>16th attempt</td><td>—</td><td>PANIC</td></tr>
</table>

<p><strong>When the 15th process calls bp2 = bread(...):</strong> <code>bget()</code> scans all 30 buffer slots. Every slot has <code>refcnt > 0</code> (held by one of the 15 processes). <code>bget()</code> finds no free slot → <code>panic("bget: no buffers")</code>.</p>

<p><strong>The fix:</strong> Always <code>brelse(bp1)</code> immediately after extracting the secondary map address — before any other <code>bread()</code> call. This limits peak buffer usage to 1 slot per bmap() call, regardless of concurrency.</p>

</div>
</div>

<div class="var-question">
<div class="var-qnum">Q11 <span class="diff diff-hard">Hard</span></div>

Write the complete <code>sys_symlink()</code> implementation. For each early <code>return -1</code>, identify what resource would leak if the cleanup call before it were removed.

<div class="answer-content">

<pre><code class="language-c">uint64
sys_symlink(void)
{
    char target[MAXPATH], path[MAXPATH];
    struct inode *ip;

    // Step 1: read string arguments from user space
    if (argstr(0, target, MAXPATH) < 0 || argstr(1, path, MAXPATH) < 0)
        return -1;
    // No cleanup needed — begin_op() not called yet

    // Step 2: start transaction
    begin_op();
    // RESOURCE: log reservation (log.outstanding++)

    // Step 3: create T_SYMLINK inode
    ip = create(path, T_SYMLINK, 0, 0);
    if (ip == 0) {
        end_op();       // ← without this: log.outstanding never decremented
        return -1;      //   → future begin_op() sleeps forever → system hangs
    }
    // RESOURCE: ip is locked (ilock held), ip->ref = 1

    // Step 4: write target path into inode's data block
    if (writei(ip, 0, (uint64)target, 0, strlen(target)) != (int)strlen(target)) {
        iunlockput(ip); // ← without this: inode lock never released + ref never decremented
        end_op();       //   → other processes can never lock this inode → deadlock
        return -1;      //   → inode slot occupied forever → "iget: no inodes" after NINODES ops
    }

    // Step 5: clean release
    iunlockput(ip);   // release inode lock, decrement ref
    end_op();         // commit transaction: inode + directory entry + data all atomic
    return 0;
}</code></pre>

<p><strong>Summary of what each cleanup releases:</strong></p>
<table>
  <tr><th>Call</th><th>Resource released</th><th>If missing: symptom</th></tr>
  <tr><td><code>end_op()</code></td><td>Log reservation</td><td>System hangs — all future file ops sleep waiting for log space</td></tr>
  <tr><td><code>iunlockput(ip)</code></td><td>Inode sleeplock + ref count</td><td>Inode permanently locked; no other process can ever use it</td></tr>
</table>

</div>
</div>
