---
layout: page
title: "Lab Predictions - lab3-a1b2"
lab: lab3
description: "Exam predictions for Lab3-w8: code tracing, wrong slot bugs, nmeta calculation, O_NOFOLLOW, and null-termination."
---

## Easy

<div class="var-question">
<div class="var-qnum">Q1 <span class="diff diff-easy">Easy</span></div>

After changing <code>FSSIZE</code> from 2000 to 200000 in <code>kernel/param.h</code>, the <code>make</code> output shows a line beginning with <code>nmeta</code>. Write out exactly what that line says and identify what changed compared to the original.

<div class="answer-content">

<p><strong>New output (FSSIZE = 200000):</strong></p>
<pre><code class="language-c">nmeta 70 (boot, super, log blocks 30 inode blocks 13, bitmap blocks 25) blocks 199930 total 200000</code></pre>

<p><strong>Original output (FSSIZE = 2000):</strong></p>
<pre><code class="language-c">nmeta 46 (boot, super, log blocks 30 inode blocks 13, bitmap blocks 1) blocks 1954 total 2000</code></pre>

<p><strong>What changed and why:</strong></p>
<table>
  <tr><th>Component</th><th>Before</th><th>After</th><th>Reason</th></tr>
  <tr><td>bitmap blocks</td><td>1</td><td>25</td><td>Each bitmap block tracks 8192 blocks. 200000 / 8192 = 24.4 → needs 25 blocks</td></tr>
  <tr><td>nmeta total</td><td>46</td><td>70</td><td>1 + 1 + 30 + 13 + 25 = 70 (+24 bitmap blocks)</td></tr>
  <tr><td>data blocks</td><td>1954</td><td>199930</td><td>200000 - 70 = 199930</td></tr>
  <tr><td>log, inode blocks</td><td>30, 13</td><td>30, 13</td><td>Unchanged — these are fixed by LOGSIZE and NINODES</td></tr>
</table>

<p><strong>Bottom line:</strong> Increasing FSSIZE only adds more bitmap blocks to track the larger disk. Log size and inode count stay the same.</p>

</div>
</div>

<div class="var-question">
<div class="var-qnum">Q2 <span class="diff diff-easy">Easy</span></div>

What does <code>O_NOFOLLOW</code> do, and why must its value (0x800) not overlap with existing flags?

<div class="answer-content">

<p><strong>What O_NOFOLLOW does:</strong></p>
<p>When passed to <code>open()</code>, it tells the kernel: "If the path resolves to a symlink, open the symlink file itself — do not follow the link to the target." Without it, <code>open("mylink")</code> transparently follows the symlink.</p>

<pre><code class="language-c">// Without O_NOFOLLOW:
fd = open("link_to_sh", O_RDONLY);    // opens /bin/sh itself

// With O_NOFOLLOW:
fd = open("link_to_sh", O_RDONLY | O_NOFOLLOW);  // opens the symlink inode itself
// reading fd gives you the path string "/bin/sh", not sh's contents</code></pre>

<p><strong>Why 0x800 must not overlap:</strong></p>
<p>The flags are combined with bitwise OR. If two flags share a bit, you cannot distinguish them:</p>
<pre><code class="language-c">// Existing flags in kernel/fcntl.h:
#define O_RDONLY   0x000   // 0000 0000 0000
#define O_WRONLY   0x001   // 0000 0000 0001
#define O_RDWR     0x002   // 0000 0000 0010
#define O_CREATE   0x200   // 0010 0000 0000
#define O_TRUNC    0x400   // 0100 0000 0000
#define O_NOFOLLOW 0x800   // 1000 0000 0000  ← safe, unique bit</code></pre>

<p>If <code>O_NOFOLLOW</code> were 0x001 (same as O_WRONLY), then <code>open("link", O_WRONLY)</code> would incorrectly trigger the no-follow behaviour. Using the next unused bit (0x800) ensures <code>omode & O_NOFOLLOW</code> only fires when explicitly set.</p>

</div>
</div>

<div class="var-question">
<div class="var-qnum">Q3 <span class="diff diff-easy">Easy</span></div>

List all 8 files you must modify for Task 2 (symbolic links) and state the single change made to each.

<div class="answer-content">

<table>
  <tr><th>File</th><th>Change</th></tr>
  <tr><td><code>kernel/stat.h</code></td><td>Add <code>#define T_SYMLINK 4</code></td></tr>
  <tr><td><code>kernel/fcntl.h</code></td><td>Add <code>#define O_NOFOLLOW 0x800</code></td></tr>
  <tr><td><code>kernel/syscall.h</code></td><td>Add <code>#define SYS_symlink &lt;N&gt;</code> (next unused syscall number)</td></tr>
  <tr><td><code>kernel/syscall.c</code></td><td>Add <code>extern uint64 sys_symlink(void);</code> and <code>[SYS_symlink] sys_symlink,</code> in the dispatch table</td></tr>
  <tr><td><code>kernel/sysfile.c</code></td><td>Add <code>sys_symlink()</code> function body + modify <code>sys_open()</code> to follow symlinks</td></tr>
  <tr><td><code>user/user.h</code></td><td>Add <code>int symlink(const char*, const char*);</code> prototype</td></tr>
  <tr><td><code>user/usys.pl</code></td><td>Add <code>entry("symlink");</code> to generate the RISC-V ecall stub</td></tr>
  <tr><td><code>Makefile</code></td><td>Add <code>$U/_symlinktest\</code> to compile the test binary</td></tr>
</table>

<p><strong>Why usys.pl is needed:</strong> <code>usys.pl</code> is a Perl script that auto-generates user-space assembly stubs. Each <code>entry("symlink")</code> generates a small assembly function that places the syscall number in register <code>a7</code> and executes <code>ecall</code>. Without this, user programs cannot call <code>symlink()</code>.</p>

</div>
</div>

<div class="var-question">
<div class="var-qnum">Q4 <span class="diff diff-easy">Easy</span></div>

What is the role of <code>T_SYMLINK = 4</code> in the kernel? Where is the type value stored and how does the kernel know a file is a symlink when it opens it?

<div class="answer-content">

<p><strong>Where the type is stored:</strong></p>
<p><code>T_SYMLINK = 4</code> is stored in the <code>type</code> field of both the on-disk <code>struct dinode</code> and in-memory <code>struct inode</code>:</p>

<pre><code class="language-c">struct dinode {
    short type;   // ← T_SYMLINK = 4 stored here on disk
    short major;
    short minor;
    short nlink;
    uint  size;
    uint  addrs[NDIRECT + 2];
};</code></pre>

<p><strong>How the kernel checks it:</strong></p>
<p>When <code>sys_open()</code> calls <code>ilock(ip)</code>, the inode is loaded from disk. The kernel then checks:</p>
<pre><code class="language-c">while (ip->type == T_SYMLINK && !(omode & O_NOFOLLOW)) {
    // follow the link...
}</code></pre>

<p><strong>The 4 type values:</strong></p>
<table>
  <tr><th>Constant</th><th>Value</th><th>Meaning</th></tr>
  <tr><td><code>T_DIR</code></td><td>1</td><td>Directory</td></tr>
  <tr><td><code>T_FILE</code></td><td>2</td><td>Regular file</td></tr>
  <tr><td><code>T_DEVICE</code></td><td>3</td><td>Device file</td></tr>
  <tr><td><code>T_SYMLINK</code></td><td>4</td><td>Symbolic link (new in Lab 3)</td></tr>
</table>

<p>A type value of 0 means the inode is free. <code>ialloc()</code> scans for type == 0 to find an unused inode slot.</p>

</div>
</div>

---

## Medium

<div class="var-question">
<div class="var-qnum">Q5 <span class="diff diff-med">Medium</span></div>

A student writes the doubly-indirect case in <code>bmap()</code> but accidentally uses <code>ip->addrs[NDIRECT]</code> (slot index 11, the singly-indirect slot) instead of <code>ip->addrs[NDIRECT+1]</code> (slot index 12) for the master map. Trace what happens when the program writes logical blocks 12 through 300.

<div class="answer-content">

<p><strong>The bug:</strong></p>
<pre><code class="language-c">// WRONG — uses the singly-indirect slot for the master map:
if ((addr = ip->addrs[NDIRECT]) == 0) {   // ← should be NDIRECT+1
    addr = balloc(ip->dev);
    ip->addrs[NDIRECT] = addr;             // ← corrupts the singly-indirect pointer!
}</code></pre>

<p><strong>Trace through writes:</strong></p>
<ol>
  <li><strong>Blocks 0–10 (direct):</strong> Work correctly — direct path uses <code>ip->addrs[bn]</code></li>
  <li><strong>Blocks 11–266 (singly-indirect):</strong> <code>bmap()</code> enters the singly-indirect case, sets <code>ip->addrs[NDIRECT]</code> to the single-level map block address. Works correctly.</li>
  <li><strong>Block 267 (first doubly-indirect):</strong>
    <ul>
      <li><code>bn -= NDIRECT</code> → bn = 256</li>
      <li><code>bn -= NINDIRECT</code> → bn = 0</li>
      <li>Enters doubly-indirect case. Reads <code>ip->addrs[NDIRECT]</code> — but this is the <strong>singly-indirect map block</strong>, not empty!</li>
      <li>Treats the singly-indirect map block as the master map</li>
      <li><code>a[0]</code> of the singly-indirect map block contains the address of data block 11 (the first singly-indirect data block)</li>
      <li>Uses data block 11's address as a secondary map address → <code>bread(data_block_11)</code> and treats file data as a pointer array</li>
      <li>File data is arbitrary bytes → returns garbage as a block number → reads/writes to a wrong physical block, or panics if the garbage block number is out of range</li>
    </ul>
  </li>
</ol>

<p><strong>Visible symptom:</strong> <code>bigfile</code> either panics the kernel or writes 6580 blocks but reads back corrupted data. The singly-indirect data blocks (11–266) become corrupted because the doubly-indirect code is overwriting their content when it thinks it's writing to secondary map blocks.</p>

<p><strong>Fix:</strong> Use <code>ip->addrs[NDIRECT+1]</code> (index 12) for the doubly-indirect master map, and ensure both <code>struct inode</code> (file.h) and <code>struct dinode</code> (fs.h) have <code>addrs[NDIRECT+2]</code>.</p>

</div>
</div>

<div class="var-question">
<div class="var-qnum">Q6 <span class="diff diff-med">Medium</span></div>

Why must the symlink-following code in <code>sys_open()</code> be placed <strong>after</strong> <code>ilock(ip)</code> but <strong>before</strong> the directory check? What goes wrong if you swap the order?

<div class="answer-content">

<p><strong>Correct placement:</strong></p>
<pre><code class="language-c">ilock(ip);                                // 1. lock first

while (ip->type == T_SYMLINK ...) {       // 2. follow symlinks
    // ...
}

if (ip->type == T_DIR && omode != O_RDONLY) {  // 3. dir check last
    iunlockput(ip); end_op(); return -1;
}</code></pre>

<p><strong>Why AFTER ilock():</strong></p>
<p><code>ilock()</code> loads the inode from disk (sets <code>ip->valid = 1</code>) and acquires the sleeplock. Without <code>ilock()</code>:</p>
<ul>
  <li><code>ip->type</code> may be 0 (stale, not loaded) → the while condition never triggers even for a real symlink</li>
  <li><code>readi()</code> requires the inode lock to safely access data blocks — calling it without the lock is a data race</li>
</ul>

<p><strong>Why BEFORE the directory check:</strong></p>
<p>Consider a symlink pointing to a directory: <code>mylink → /home/user</code></p>
<pre><code class="language-c">// If directory check runs FIRST (WRONG):
ip = namei("mylink");    // ip->type = T_SYMLINK
ilock(ip);
// Directory check: T_SYMLINK != T_DIR → passes (no error)
// Symlink follow: ip = /home/user (T_DIR)
// Now T_DIR check never runs on the actual directory!
// open("mylink", O_WRONLY) succeeds — you can write to a directory!

// Correct order: follow first, then check:
// After following: ip = /home/user (T_DIR)
// Directory check: T_DIR && O_WRONLY → return -1 (correct!)</code></pre>

<p>Placing the symlink follow before the directory check ensures the directory check applies to the <strong>final resolved inode</strong>, not the symlink itself.</p>

</div>
</div>

<div class="var-question">
<div class="var-qnum">Q7 <span class="diff diff-med">Medium</span></div>

Trace the execution of <code>bmap(ip, 523)</code> step by step through the doubly-indirect code. Show every subtraction, division, and modulo operation.

<div class="answer-content">

<p><strong>Starting value: bn = 523</strong></p>

<pre><code class="language-c">// Step 1: Is it direct? (bn < NDIRECT = 11?)
523 < 11?  → NO

// Step 2: Subtract direct range
bn -= NDIRECT;   // bn = 523 - 11 = 512

// Step 3: Is it singly-indirect? (bn < NINDIRECT = 256?)
512 < 256?  → NO

// Step 4: Subtract singly-indirect range
bn -= NINDIRECT;  // bn = 512 - 256 = 256

// Step 5: Is it doubly-indirect? (bn < NDINDIRECT = 65536?)
256 < 65536?  → YES — enter doubly-indirect code

// Step 6: Calculate indices
idx1 = bn / NINDIRECT = 256 / 256 = 1   // → Secondary Map #1
idx2 = bn % NINDIRECT = 256 % 256 = 0   // → slot 0 in that map</code></pre>

<p><strong>Navigation:</strong></p>
<ol>
  <li>Load master map block from <code>ip->addrs[12]</code></li>
  <li>Look at <code>master_map[idx1=1]</code> → address of Secondary Map #1</li>
  <li><code>brelse(master_map)</code> — release before loading next level</li>
  <li>Load Secondary Map #1</li>
  <li>Look at <code>secondary_map_1[idx2=0]</code> → address of the actual data block</li>
  <li><code>brelse(secondary_map_1)</code></li>
  <li>Return the data block address</li>
</ol>

<p><strong>Context — what logical block 523 represents:</strong></p>
<table>
  <tr><th>Range</th><th>Type</th><th>Blocks</th></tr>
  <tr><td>0 – 10</td><td>Direct</td><td>11 blocks</td></tr>
  <tr><td>11 – 266</td><td>Singly-indirect</td><td>256 blocks</td></tr>
  <tr><td>267 – 522</td><td>Doubly-indirect (secondary map 0)</td><td>256 blocks</td></tr>
  <tr><td><strong>523</strong></td><td><strong>Doubly-indirect (secondary map 1, slot 0) ← block 523</strong></td><td></td></tr>
</table>

<p>Block 523 is the <strong>very first block in secondary map #1</strong> (idx1=1, idx2=0). This is a good boundary-condition test case.</p>

</div>
</div>

<div class="var-question">
<div class="var-qnum">Q8 <span class="diff diff-med">Medium</span></div>

In <code>sys_symlink()</code>, after calling <code>writei()</code> to store the target path, a student forgets to null-terminate the string. They write <code>strlen(target)</code> bytes without a <code>\0</code>. What happens when <code>sys_open()</code> later tries to follow this symlink?

<div class="answer-content">

<p><strong>The writei call in sys_symlink():</strong></p>
<pre><code class="language-c">// Writes exactly strlen(target) bytes — no null terminator
writei(ip, 0, (uint64)target, 0, strlen(target));</code></pre>

<p><strong>The readi call in sys_open():</strong></p>
<pre><code class="language-c">char target[MAXPATH];
int n = readi(ip, 0, (uint64)target, 0, MAXPATH - 1);
target[n] = '\0';   // ← THIS line saves us — we null-terminate after reading</code></pre>

<p><strong>Why it still works if you null-terminate after readi():</strong></p>
<p>The solution code always does <code>target[n] = '\0'</code> after reading. Since <code>n</code> = number of bytes actually written = <code>strlen(original_path)</code>, this places a null byte at exactly the right position. So even though the stored bytes have no null terminator on disk, the in-memory <code>target[]</code> buffer is correctly terminated.</p>

<p><strong>What happens if you ALSO forget target[n] = '\0':</strong></p>
<pre><code class="language-c">// Without null-termination:
char target[MAXPATH];   // stack memory — uninitialized garbage after n bytes
int n = readi(ip, 0, (uint64)target, 0, MAXPATH - 1);
// target[0..n-1] = correct path bytes
// target[n..MAXPATH-2] = whatever was on the stack (garbage)
// NO null terminator added

ip = namei(target);   // namei reads until '\0'
// → reads past the actual path into garbage bytes
// → path resolution fails (garbage path not found) → namei returns 0
// → open() returns -1 (dangling link error)</code></pre>

<p><strong>Best practice:</strong> Always <code>target[n] = '\0'</code> immediately after any <code>readi()</code> that reads a path string. <code>readi()</code> never adds a null terminator — it is a raw byte copy.</p>

</div>
</div>

---

## Hard

<div class="var-question">
<div class="var-qnum">Q9 <span class="diff diff-hard">Hard</span></div>

The solution's <code>itrunc()</code> doubly-indirect section calls <code>brelse(bp2)</code> before <code>bfree(ip->dev, a1[i])</code>. A student reverses this order, calling <code>bfree</code> first. Explain precisely why this is dangerous and describe a scenario where it causes corruption.

<div class="answer-content">

<p><strong>The correct order:</strong></p>
<pre><code class="language-c">brelse(bp2);              // 1. release the buffer FIRST
bfree(ip->dev, a1[i]);   // 2. mark block as free SECOND</code></pre>

<p><strong>The buggy order:</strong></p>
<pre><code class="language-c">bfree(ip->dev, a1[i]);   // 1. mark block as free FIRST
brelse(bp2);              // 2. release buffer SECOND (too late!)</code></pre>

<p><strong>Why the buggy order is dangerous:</strong></p>
<p><code>bfree()</code> calls <code>bread()</code> internally to load the bitmap block and clear the bit. After <code>bfree()</code> returns, the secondary map block is marked <strong>free in the bitmap</strong>. At this point, another process running concurrently could call <code>balloc()</code>, find that block available, and start using it for a new file — while <code>bp2</code> is still locked in the buffer cache by the current process.</p>

<p><strong>Concrete corruption scenario:</strong></p>
<ol>
  <li>Process A: deletes a large file. <code>bfree(dev, secondary_map_block_42)</code> marks block 42 as free</li>
  <li>Process B: creates a new file, calls <code>balloc()</code> → finds block 42 free → allocates it → calls <code>bzero(dev, 42)</code> → overwrites block 42 with zeros</li>
  <li>Process A: still holds <code>bp2</code> pointing to block 42's buffer. <code>brelse(bp2)</code> releases it. The buffer cache now has a stale/zeroed copy of block 42</li>
  <li>Process B continues writing file data to block 42 via the same buffer cache entry</li>
  <li>The buffer cache now has two conflicting views of block 42 — corruption is guaranteed</li>
</ol>

<p><strong>The invariant:</strong> A block must be released from the buffer cache (<code>brelse</code>) before it is returned to the free pool (<code>bfree</code>). Otherwise the block can be reallocated while still in use.</p>

</div>
</div>

<div class="var-question">
<div class="var-qnum">Q10 <span class="diff diff-hard">Hard</span></div>

Write the complete <code>sys_symlink()</code> implementation including all error paths. For each early return, explain what resource would leak if you forgot to call the corresponding cleanup function.

<div class="answer-content">

<p><strong>Complete implementation with annotations:</strong></p>
<pre><code class="language-c">uint64
sys_symlink(void)
{
    char target[MAXPATH], path[MAXPATH];
    struct inode *ip;

    // ── Step 1: Get string arguments from user space ──
    if (argstr(0, target, MAXPATH) < 0 || argstr(1, path, MAXPATH) < 0)
        return -1;
    // No cleanup needed: begin_op() not called yet, no resources allocated

    // ── Step 2: Start transaction ──
    begin_op();
    // RESOURCE: log reservation. Must call end_op() on ALL subsequent returns.

    // ── Step 3: Create T_SYMLINK inode ──
    ip = create(path, T_SYMLINK, 0, 0);
    if (ip == 0) {
        end_op();    // ← required: releases log reservation
        return -1;
    }
    // RESOURCE: ip is locked (create() returns with ilock held)
    //           ip->ref++ was called, must call iunlockput() to release

    // ── Step 4: Write target path into inode data blocks ──
    if (writei(ip, 0, (uint64)target, 0, strlen(target)) != (int)strlen(target)) {
        iunlockput(ip);   // ← required: releases inode lock + decrements ref
        end_op();         // ← required: releases log reservation
        return -1;
    }

    // ── Step 5: Clean up normally ──
    iunlockput(ip);   // releases lock, decrements ref
    end_op();         // commits transaction to disk
    return 0;
}</code></pre>

<p><strong>What leaks if each cleanup is omitted:</strong></p>
<table>
  <tr><th>Missing call</th><th>What leaks</th><th>Symptom</th></tr>
  <tr><td><code>end_op()</code> after create fails</td><td>Log reservation: <code>log.outstanding</code> never decremented</td><td>Future <code>begin_op()</code> calls sleep forever waiting for log space → system hangs</td></tr>
  <tr><td><code>iunlockput(ip)</code> after writei fails</td><td>Inode lock not released + ref count stays at 1</td><td>No other process can ever lock this inode → deadlock; inode slot permanently occupied</td></tr>
  <tr><td><code>iunlockput(ip)</code> on success path</td><td>Same — inode ref = 1 permanently</td><td>inode table exhausted after NINODES (200) such leaks → panic "iget: no inodes"</td></tr>
</table>

</div>
</div>

<div class="var-question">
<div class="var-qnum">Q11 <span class="diff diff-hard">Hard</span></div>

The <code>bmap()</code> doubly-indirect code calls <code>log_write(bp)</code> after updating <code>a[idx1]</code> in the master map, then calls <code>brelse(bp)</code> before loading the secondary map. If the machine crashes between <code>log_write(bp)</code> and the <code>end_op()</code> that commits the transaction, is the filesystem left in a consistent state? Explain using the write-ahead log protocol.

<div class="answer-content">

<p><strong>The sequence in question:</strong></p>
<pre><code class="language-c">// Inside bmap() during a write():
a[idx1] = addr;          // step 1: update master map in RAM (buffer)
log_write(bp);           // step 2: mark this buffer as part of the pending transaction
brelse(bp);              // step 3: release (but buffer stays pinned in cache)
// ... more work ...
// --- CRASH HERE? ---
// end_op() inside writei() → filewrite() commits everything</code></pre>

<p><strong>What log_write() actually does:</strong></p>
<p><code>log_write(bp)</code> does NOT write to disk immediately. It:</p>
<ol>
  <li>Marks the buffer with <code>B_DIRTY</code></li>
  <li>Records the buffer's block number in the in-memory log header (<code>log.lh.block[]</code>)</li>
  <li>Pins the buffer in cache (increments refcnt to prevent eviction)</li>
</ol>
<p>The actual disk write happens only when <code>end_op()</code> triggers <code>commit()</code>.</p>

<p><strong>If crash occurs before end_op():</strong></p>
<pre><code class="language-c">// On reboot, initlog() calls recover_from_log():
// Reads the log header from disk
// If log.lh.n == 0: no committed transaction → nothing to replay
// The master map update (in RAM only) was NEVER written to disk
// The filesystem is in the SAME state as before writei() was called
// Result: CONSISTENT — the partial write is silently discarded</code></pre>

<p><strong>If crash occurs AFTER end_op() but before install_trans():</strong></p>
<pre><code class="language-c">// On reboot: log header shows n > 0 (commit record was written)
// recover_from_log() replays all logged blocks to their real locations
// The master map, secondary map, and data blocks are all written correctly
// Result: CONSISTENT — full write is replayed atomically</code></pre>

<p><strong>The invariant the WAL provides:</strong> Either ALL blocks in a transaction reach their final disk locations (if committed), or NONE do (if not committed). There is no in-between state. The master map update, secondary map update, data block write, and inode size update all commit together or not at all.</p>

</div>
</div>
