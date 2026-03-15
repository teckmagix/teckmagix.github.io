---
layout: page
title: "Lab Predictions - lab3-w9m5"
lab: lab3
description: "Exam predictions for Lab3-w8: boundary blocks, usys.pl, T_SYMLINK type, inode table, and concurrent symlink test."
---

## Easy

<div class="var-question">
<div class="var-qnum">Q1 <span class="diff diff-easy">Easy</span></div>

What does <code>user/usys.pl</code> do and what does adding <code>entry("symlink")</code> generate?

<div class="answer-content">

<p><strong>What usys.pl is:</strong></p>
<p><code>usys.pl</code> is a Perl script that generates a small assembly file (<code>usys.S</code>) at build time. For each <code>entry("name")</code> call, it produces a short assembly function that user programs can call to invoke a system call. The build system compiles this assembly file and links it into every user program.</p>

<p><strong>What entry("symlink") generates:</strong></p>
<pre><code class="language-asm">symlink:          # function name — called from C via symlink(target, path)
    li a7, SYS_symlink    # load system call number (22) into register a7
    ecall                 # trigger the RISC-V trap — jump to kernel
    ret                   # return to caller with kernel's return value in a0</code></pre>

<p><strong>The full user-to-kernel path for symlink():</strong></p>

<table>
  <tr><th>Layer</th><th>File</th><th>What happens</th></tr>
  <tr><td>1. C call</td><td><code>user/user.h</code></td><td>Compiler resolves <code>symlink(target, path)</code> to the assembly stub</td></tr>
  <tr><td>2. Assembly stub</td><td><code>user/usys.S</code> (generated)</td><td>Loads 22 into a7, executes <code>ecall</code></td></tr>
  <tr><td>3. Trap entry</td><td><code>kernel/trampoline.S</code></td><td>Saves user registers, switches to kernel page table</td></tr>
  <tr><td>4. Dispatch</td><td><code>kernel/syscall.c</code></td><td>Reads a7=22, calls <code>syscalls[22]</code> = <code>sys_symlink</code></td></tr>
  <tr><td>5. Handler</td><td><code>kernel/sysfile.c</code></td><td>Runs <code>sys_symlink()</code>, returns result in a0</td></tr>
</table>

<p>Without the <code>entry("symlink")</code> line, the stub function is never generated. A user program calling <code>symlink()</code> would get a linker error: <code>undefined reference to 'symlink'</code>.</p>

</div>
</div>

<div class="var-question">
<div class="var-qnum">Q2 <span class="diff diff-easy">Easy</span></div>

Why is <code>T_SYMLINK</code> defined as 4 specifically? What are the other type constants and where are they stored?

<div class="answer-content">

<p><strong>All inode type constants:</strong></p>
<pre><code class="language-c">// kernel/stat.h
#define T_DIR     1   // Directory
#define T_FILE    2   // Regular file
#define T_DEVICE  3   // Device file
#define T_SYMLINK 4   // Symbolic link  ← added in this lab</code></pre>

<p><strong>Why 4:</strong></p>
<p>4 is simply the next unused integer. These constants are stored in the <code>type</code> field of <code>struct dinode</code> (on disk, 2-byte <code>short</code>) and <code>struct inode</code> (in memory). The values must be unique so the kernel can distinguish inode types with a simple equality check:</p>
<pre><code class="language-c">if(ip->type == T_SYMLINK) { ... }   // unique value — no ambiguity</code></pre>

<p><strong>Where the type is checked throughout the kernel:</strong></p>

<table>
  <tr><th>Location</th><th>Check</th><th>Purpose</th></tr>
  <tr><td><code>sys_open()</code></td><td><code>ip->type == T_SYMLINK</code></td><td>Follow the symlink instead of opening directly</td></tr>
  <tr><td><code>sys_open()</code></td><td><code>ip->type == T_DIR</code></td><td>Prevent write-opening a directory</td></tr>
  <tr><td><code>sys_mkdir()</code>, <code>sys_mknod()</code></td><td>uses <code>create(..., T_DIR, ...)</code></td><td>Create a directory inode</td></tr>
  <tr><td><code>dirlookup()</code></td><td>checks <code>T_DIR</code> on parent</td><td>Ensure parent is a directory before searching its entries</td></tr>
  <tr><td><code>filestat()</code></td><td>copies <code>ip->type</code> to <code>struct stat</code></td><td>Returns type to user via <code>fstat()</code> syscall</td></tr>
</table>

</div>
</div>

<div class="var-question">
<div class="var-qnum">Q3 <span class="diff diff-easy">Easy</span></div>

What does <code>symlinktest</code> test with its <em>concurrent symlinks</em> test? Why does concurrency add complexity beyond the basic test?

<div class="answer-content">

<p><strong>What the two tests check:</strong></p>

<table>
  <tr><th>Test</th><th>What it verifies</th></tr>
  <tr><td><code>test symlinks</code></td><td>Basic correctness: create a symlink, open through it, verify data, test <code>O_NOFOLLOW</code>, test dangling link, test depth limit</td></tr>
  <tr><td><code>test concurrent symlinks</code></td><td>Multiple processes simultaneously creating and opening symlinks — verifies that the implementation is safe under concurrency</td></tr>
</table>

<p><strong>Why concurrency adds complexity:</strong></p>
<p>The symlink creation path in <code>sys_symlink()</code> is wrapped in <code>begin_op()</code>/<code>end_op()</code>, making disk writes atomic. But concurrent calls mean multiple processes are simultaneously:</p>
<ol>
  <li>Calling <code>balloc()</code> to allocate inode blocks — races on the bitmap</li>
  <li>Calling <code>create()</code> which locks directory entries — races on directory inodes</li>
  <li>Calling <code>bread()</code> on the same or nearby blocks — races on the buffer cache</li>
</ol>

<p>If any of these is not correctly serialised by the kernel's existing locking, two processes could both see a "free" inode slot and both try to use it — resulting in two files sharing one inode, corrupting both. The concurrent test exposes such bugs by running many symlink operations simultaneously and checking that all results are consistent.</p>

<p><strong>Expected passing output:</strong></p>
<pre><code class="language-c">Start: test symlinks
test symlinks: ok
Start: test concurrent symlinks
test concurrent symlinks: ok</code></pre>

</div>
</div>

<div class="var-question">
<div class="var-qnum">Q4 <span class="diff diff-easy">Easy</span></div>

What does the <code>nmeta</code> line printed by <code>mkfs</code> tell you, and what changed between the original and the Lab 3 filesystem?

<div class="answer-content">

<p><strong>The nmeta output format:</strong></p>
<pre><code class="language-c">// After changing FSSIZE to 200000:
nmeta 70 (boot, super, log blocks 30 inode blocks 13, bitmap blocks 25) blocks 199930 total 200000
//   ^^  ^^ ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^         ^^^^^^        ^^^^^^
//   |   |   Description of metadata blocks                                        Data blocks  Total</code></pre>

<p><strong>What each number means:</strong></p>

<table>
  <tr><th>Field</th><th>Original (FSSIZE=2000)</th><th>Lab 3 (FSSIZE=200000)</th></tr>
  <tr><td>boot</td><td>1 block</td><td>1 block</td></tr>
  <tr><td>super</td><td>1 block</td><td>1 block</td></tr>
  <tr><td>log blocks</td><td>30 blocks</td><td>30 blocks</td></tr>
  <tr><td>inode blocks</td><td>13 blocks</td><td>13 blocks</td></tr>
  <tr><td>bitmap blocks</td><td>1 block</td><td>25 blocks</td></tr>
  <tr><td>nmeta total</td><td>46 blocks</td><td>70 blocks</td></tr>
  <tr><td>data blocks</td><td>1,954 blocks</td><td>199,930 blocks</td></tr>
  <tr><td>FSSIZE</td><td>2,000</td><td>200,000</td></tr>
</table>

<p><strong>Why bitmap blocks increased from 1 to 25:</strong></p>
<p>The free-space bitmap has one bit per disk block. With FSSIZE=2000: 2000 bits = 250 bytes — fits in 1 block (1024 bytes). With FSSIZE=200000: 200,000 bits = 25,000 bytes — requires ⌈25000/1024⌉ = 25 blocks. The bitmap grows proportionally with the disk size.</p>

<p><strong>Why inode blocks stayed at 13:</strong></p>
<p>The number of inodes (NINODES in <code>param.h</code>) was not changed — still 200 inodes. 13 blocks of 1 KB each, with each dinode being 64 bytes, holds ⌊13 × 1024 / 64⌋ = 208 inodes — more than enough for the tests.</p>

</div>
</div>

---

## Medium

<div class="var-question">
<div class="var-qnum">Q5 <span class="diff diff-med">Medium</span></div>

Given <code>NDIRECT=11</code> and <code>NINDIRECT=256</code>, which logical block numbers fall in each of the three ranges (direct, singly-indirect, doubly-indirect)? Give the first and last block number for each range.

<div class="answer-content">

<p><strong>Range boundaries:</strong></p>

<table>
  <tr><th>Range</th><th>First block (bn=)</th><th>Last block (bn=)</th><th>Count</th><th>Accessed via</th></tr>
  <tr><td>Direct</td><td>0</td><td>10</td><td>11</td><td><code>ip->addrs[0..10]</code></td></tr>
  <tr><td>Singly-indirect</td><td>11</td><td>266</td><td>256</td><td><code>ip->addrs[11]</code> → indirect block → data</td></tr>
  <tr><td>Doubly-indirect</td><td>267</td><td>65,802</td><td>65,536</td><td><code>ip->addrs[12]</code> → master → secondary → data</td></tr>
</table>

<p><strong>How to derive the boundaries:</strong></p>
<pre><code class="language-c">// Direct range: bn = 0 to NDIRECT-1
First = 0
Last  = NDIRECT - 1 = 11 - 1 = 10

// Singly-indirect range: bn = NDIRECT to NDIRECT+NINDIRECT-1
First = NDIRECT             = 11
Last  = NDIRECT + NINDIRECT - 1 = 11 + 256 - 1 = 266

// Doubly-indirect range: bn = NDIRECT+NINDIRECT to NDIRECT+NINDIRECT+NDINDIRECT-1
First = NDIRECT + NINDIRECT             = 11 + 256       = 267
Last  = NDIRECT + NINDIRECT + NDINDIRECT - 1 = 267 + 65536 - 1 = 65802

// MAXFILE = NDIRECT + NINDIRECT + NDINDIRECT = 11 + 256 + 65536 = 65803
// So valid logical block numbers are 0 to 65802 (65803 total)
// bn = 65803 would trigger: panic("bmap: out of range")</code></pre>

<p><strong>Boundary cases to know for the exam:</strong></p>

<table>
  <tr><th>bn</th><th>Range</th><th>Note</th></tr>
  <tr><td>0</td><td>Direct</td><td>First direct block</td></tr>
  <tr><td>10</td><td>Direct</td><td>Last direct block</td></tr>
  <tr><td>11</td><td>Singly-indirect</td><td>First use of indirect block (addrs[11])</td></tr>
  <tr><td>266</td><td>Singly-indirect</td><td>Last singly-indirect block</td></tr>
  <tr><td>267</td><td>Doubly-indirect</td><td>First use of master map (addrs[12]); idx1=0, idx2=0</td></tr>
  <tr><td>65802</td><td>Doubly-indirect</td><td>Last valid block; idx1=255, idx2=255</td></tr>
  <tr><td>65803</td><td>Out of range</td><td>panic("bmap: out of range")</td></tr>
</table>

</div>
</div>

<div class="var-question">
<div class="var-qnum">Q6 <span class="diff diff-med">Medium</span></div>

<code>SYS_symlink</code> is defined as 22. Why does this number matter, and what breaks if you use 21 (which is already taken by <code>SYS_close</code>)?

<div class="answer-content">

<p><strong>How system call dispatch works:</strong></p>
<pre><code class="language-c">// kernel/syscall.c
static uint64 (*syscalls[])(void) = {
[SYS_fork]    sys_fork,    // 1
[SYS_exit]    sys_exit,    // 2
// ...
[SYS_close]   sys_close,   // 21
[SYS_symlink] sys_symlink, // 22
};

// When ecall fires with a7 = N:
void syscall(void){
    int num = p->trapframe->a7;   // read system call number
    if(num > 0 &amp;&amp; num &lt; NELEM(syscalls) &amp;&amp; syscalls[num]) {
        p->trapframe->a0 = syscalls[num]();   // call syscalls[N]
    }
}</code></pre>

<p><strong>What happens if SYS_symlink = 21 (collision with SYS_close):</strong></p>
<pre><code class="language-c">// kernel/syscall.c — both entries try to use index 21:
[SYS_close]   sys_close,    // [21] = sys_close
[SYS_symlink] sys_symlink,  // [21] = sys_symlink  ← overwrites!

// C designated initialiser: later assignment wins.
// Result: syscalls[21] = sys_symlink  (sys_close is gone)

// Effect 1: close(fd) now calls sys_symlink() instead of sys_close().
//   sys_symlink() calls argstr(0, target, MAXPATH) — tries to read fd (an int) as a string pointer.
//   Likely returns -1 immediately. File descriptor is never closed → resource leak.

// Effect 2: symlink(target, path) calls sys_symlink() via a7=21 — appears to work.
//   But now close() is permanently broken for every process.

// Effect 3: Any process that closes a file descriptor (including stdout/stderr on exit)
//   fails silently or crashes — corrupting the open file table over time.</code></pre>

<p><strong>Why 22 is correct:</strong></p>
<p>22 is the next integer after 21 (<code>SYS_close</code>), and no existing system call uses it. Each system call number is a unique index into the <code>syscalls[]</code> array. Uniqueness is the only requirement — the specific value 22 has no special meaning beyond being available.</p>

</div>
</div>

<div class="var-question">
<div class="var-qnum">Q7 <span class="diff diff-med">Medium</span></div>

What is the purpose of <code>itrunc()</code>'s <code>ip-&gt;addrs[NDIRECT+1] = 0</code> line, and where does the in-memory inode get written back to the on-disk inode after this change?

<div class="answer-content">

<p><strong>What itrunc() does to the inode after freeing blocks:</strong></p>
<pre><code class="language-c">// End of doubly-indirect cleanup in itrunc():
brelse(bp1);
bfree(ip->dev, ip->addrs[NDIRECT+1]);
ip->addrs[NDIRECT+1] = 0;      // ← zero the slot in the IN-MEMORY inode

// Later in itrunc():
ip->size = 0;
iupdate(ip);                   // ← write the modified inode back to disk</code></pre>

<p><strong>The iupdate() call:</strong></p>
<p><code>iupdate(ip)</code> is called at the end of <code>itrunc()</code>. It copies the current in-memory inode fields — including the zeroed <code>addrs[]</code> — back to the corresponding <code>struct dinode</code> on disk. Without <code>iupdate()</code>, the in-memory change is lost on reboot and the on-disk inode still points to freed blocks.</p>

<p><strong>The full write path from in-memory to on-disk:</strong></p>
<pre><code class="language-c">// iupdate(ip):
struct buf *bp = bread(ip->dev, IBLOCK(ip->inum, sb));   // load inode block from disk
struct dinode *dip = (struct dinode*)bp->data + ip->inum % IPB;
dip->type  = ip->type;
dip->major = ip->major;
dip->minor = ip->minor;
dip->nlink = ip->nlink;
dip->size  = ip->size;
memmove(dip->addrs, ip->addrs, sizeof(ip->addrs));        // copy zeroed addrs[]
log_write(bp);                                             // mark dirty
brelse(bp);</code></pre>

<p>The <code>memmove</code> copies the zeroed <code>ip->addrs[NDIRECT+1]</code> into the on-disk <code>dinode->addrs[NDIRECT+1]</code>. On the next boot, when the inode is loaded from disk, it reads 0 for the doubly-indirect slot — it will not attempt to follow a stale pointer to a freed block.</p>

</div>
</div>

<div class="var-question">
<div class="var-qnum">Q8 <span class="diff diff-med">Medium</span></div>

What does the <code>writei()</code> call in <code>sys_symlink()</code> return, and why does the code check <code>!= strlen(target)</code> rather than just <code>&lt; 0</code>?

<div class="answer-content">

<p><strong>The check in sys_symlink():</strong></p>
<pre><code class="language-c">if(writei(ip, 0, (uint64)target, 0, strlen(target)) != strlen(target)){
    iunlockput(ip);
    end_op();
    return -1;
}</code></pre>

<p><strong>What writei() returns:</strong></p>
<p><code>writei(ip, user_src, src, off, n)</code> writes <code>n</code> bytes from address <code>src</code> into the inode's data blocks starting at byte offset <code>off</code>. It returns the number of bytes actually written. This is normally equal to <code>n</code> but can be less if the disk is full or the maximum file size is reached mid-write.</p>

<p><strong>Why <code>!= strlen(target)</code> is stricter than <code>&lt; 0</code>:</strong></p>

<table>
  <tr><th>Return value</th><th>Meaning</th><th><code>&lt; 0</code> catches it?</th><th><code>!= strlen</code> catches it?</th></tr>
  <tr><td>-1</td><td>Error (bad address, etc.)</td><td>Yes</td><td>Yes</td></tr>
  <tr><td>0</td><td>0 bytes written (disk full immediately)</td><td>No</td><td>Yes — if target non-empty</td></tr>
  <tr><td>5 (partial write)</td><td>Only 5 of 11 bytes written — disk filled mid-write</td><td>No</td><td>Yes</td></tr>
  <tr><td>11 (= strlen)</td><td>Full success</td><td>N/A</td><td>Passes check</td></tr>
</table>

<p><strong>Why a partial write is dangerous:</strong></p>
<pre><code class="language-c">// Target: "/usr/bin/sh" (11 chars)
// writei() writes only 5 bytes: "/usr/"
// symlink inode now contains "/usr/\0..." (5 bytes in data block)
// Later: open("mylink") reads 5 bytes → target = "/usr/"
// namei("/usr/") returns the /usr directory inode
// open() of a directory with O_WRONLY → error (directory check)
// But if opened O_RDONLY: opens /usr itself — completely wrong target</code></pre>

<p>Checking for an exact match with <code>strlen(target)</code> catches both total failures (-1) and partial writes (0 or fewer bytes than expected), ensuring the symlink is either fully created or not created at all.</p>

</div>
</div>

---

## Hard

<div class="var-question">
<div class="var-qnum">Q9 <span class="diff diff-hard">Hard</span></div>

Calculate <code>idx1</code> and <code>idx2</code> for logical block numbers <code>bn = 65802</code> (the last valid doubly-indirect block) and <code>bn = 523</code>. Show every subtraction step.

<div class="answer-content">

<p><strong>Constants:</strong> <code>NDIRECT=11</code>, <code>NINDIRECT=256</code></p>

<p><strong>bn = 65802 (last valid block):</strong></p>
<pre><code class="language-c">bn = 65802
bn -= NDIRECT;       // 65802 - 11 = 65791
// 65791 &lt; NINDIRECT(256)? No → not singly-indirect
bn -= NINDIRECT;     // 65791 - 256 = 65535
// 65535 &lt; NDINDIRECT(65536)? Yes → doubly-indirect

idx1 = 65535 / 256 = 255   // last secondary map (secondary map #255)
idx2 = 65535 % 256 = 255   // last slot in that secondary map

// Physical path:
// ip->addrs[12] → master_map[255] → secondary_map[255] → data block</code></pre>

<p><strong>bn = 523:</strong></p>
<pre><code class="language-c">bn = 523
bn -= NDIRECT;       // 523 - 11 = 512
// 512 &lt; NINDIRECT(256)? No → not singly-indirect
bn -= NINDIRECT;     // 512 - 256 = 256
// 256 &lt; NDINDIRECT(65536)? Yes → doubly-indirect

idx1 = 256 / 256 = 1    // second secondary map (secondary map #1)
idx2 = 256 % 256 = 0    // first slot of secondary map #1

// Physical path:
// ip->addrs[12] → master_map[1] → secondary_map[0] → data block</code></pre>

<p><strong>What these mean structurally:</strong></p>

<table>
  <tr><th>bn</th><th>After - NDIRECT</th><th>After - NINDIRECT</th><th>idx1</th><th>idx2</th><th>Secondary map</th></tr>
  <tr><td>267</td><td>256</td><td>0</td><td>0</td><td>0</td><td>Map #0, slot 0 (first)</td></tr>
  <tr><td>522</td><td>511</td><td>255</td><td>0</td><td>255</td><td>Map #0, slot 255 (last in map #0)</td></tr>
  <tr><td>523</td><td>512</td><td>256</td><td>1</td><td>0</td><td>Map #1, slot 0 (first in map #1)</td></tr>
  <tr><td>65802</td><td>65791</td><td>65535</td><td>255</td><td>255</td><td>Map #255, slot 255 (absolute last)</td></tr>
</table>

<p>Block 267 through 522 all share secondary map #0 (idx1=0) — they require only one allocation of the secondary map block and one subsequent load of it. Block 523 requires a new secondary map block to be allocated (secondary map #1). Each secondary map covers exactly 256 consecutive logical blocks.</p>

</div>
</div>

<div class="var-question">
<div class="var-qnum">Q10 <span class="diff diff-hard">Hard</span></div>

Suppose a student implements <code>bmap()</code> correctly but forgets to add the doubly-indirect cleanup to <code>itrunc()</code>. Describe the exact sequence of events that leads to filesystem corruption after creating and deleting multiple large files.

<div class="answer-content">

<p><strong>What happens on first large file creation and deletion:</strong></p>
<pre><code class="language-c">// Step 1: Create bigfile (TEST_BLOCKS = 6580 blocks)
// bmap() allocates:
//   - 1 master map block (e.g. disk block 500)
//   - 26 secondary map blocks (⌈6580/256⌉ = 26, e.g. blocks 501–526)
//   - 6580 data blocks (e.g. blocks 527–7106)
// Total allocated: 1 + 26 + 6580 = 6607 blocks

// Step 2: Delete bigfile — itrunc() runs WITHOUT doubly-indirect cleanup
// itrunc() correctly frees:
//   - 11 direct blocks (freed by existing direct block loop)
//   - 1 indirect block + 245 singly-indirect data blocks (freed by existing indirect loop)
//   Wait — TEST_BLOCKS = 6580:
//     - 11 direct
//     - 256 singly-indirect
//     - 6580 - 11 - 256 = 6313 doubly-indirect data blocks
// itrunc() WITHOUT fix only frees 11 + 1 + 256 = 268 blocks.
// LEAKED: 1 master map + 26 secondary maps + 6313 data blocks = 6340 blocks PERMANENTLY LOST</code></pre>

<p><strong>The bitmap corruption:</strong></p>
<p>After the buggy deletion:</p>
<ul>
  <li>The bitmap still shows blocks 500–526 and the 6313 doubly-indirect data blocks as <strong>allocated</strong></li>
  <li>No inode references them — they are orphaned</li>
  <li>These blocks are permanently excluded from <code>balloc()</code>'s search</li>
</ul>

<p><strong>Cascading failure after repeated operations:</strong></p>
<pre><code class="language-c">// After 1 create + delete cycle: ~6340 blocks leaked
// After 2 cycles: ~12,680 blocks leaked
// After N cycles: ~6340 × N blocks leaked

// With FSSIZE=200000 and 199,930 data blocks:
// After ~31 cycles: 31 × 6340 = 196,540 blocks leaked
// balloc() scans the entire bitmap, finds no free bit:
panic("balloc: out of blocks");</code></pre>

<p><strong>Symptom visible to the user:</strong></p>
<p>The first ~30 runs of <code>bigfile</code> succeed. Then suddenly:</p>
<pre><code class="language-c">$ bigfile
....
wrote 267 blocks      ← stops exactly at NDIRECT + NINDIRECT = 267
bigfile: file is too small   ← balloc() returned 0 at block 268 (first doubly-indirect)</code></pre>

<p>The root cause: every block in the doubly-indirect range is marked "allocated" in the bitmap from the previous leaked files — <code>balloc()</code> skips them all and runs out of space.</p>

</div>
</div>

<div class="var-question">
<div class="var-qnum">Q11 <span class="diff diff-hard">Hard</span></div>

A student adds the symlink loop in <code>sys_open()</code> but places it <em>before</em> <code>ilock(ip)</code> instead of after. Identify every bug this introduces and explain why each one is dangerous.

<div class="answer-content">

<p><strong>The buggy placement:</strong></p>
<pre><code class="language-c">} else {
    if((ip = namei(path)) == 0){
        end_op();
        return -1;
    }
    // BUG: symlink loop placed BEFORE ilock
    int depth = 0;
    while(ip->type == T_SYMLINK &amp;&amp; !(omode &amp; O_NOFOLLOW)){  // ← reads type WITHOUT lock
        ...
        readi(ip, 0, (uint64)target, 0, MAXPATH);            // ← reads data WITHOUT lock
        ...
    }
    ilock(ip);          // lock acquired too late
    if(ip->type == T_DIR &amp;&amp; omode != O_RDONLY){ ... }
}</code></pre>

<p><strong>Bug 1 — Reading ip->type without a lock (TOCTOU race):</strong></p>
<pre><code class="language-c">// Thread A: ip->type == T_SYMLINK  → true, enters loop
// Thread B: itrunc(ip) runs, sets ip->type = 0 (freed)
// Thread A: readi(ip, ...) — reads data from a freed/reused inode
// Data read is garbage. namei(garbage) returns 0. sys_open returns -1.
// Or worse: the inode was reallocated as T_FILE — readi reads a real file's data
// as a path string — namei follows a nonsense path.</code></pre>

<p><strong>Bug 2 — readi() without holding the inode lock:</strong></p>
<p><code>readi()</code> internally reads the inode's size field and block addresses. Reading these without the lock means another process could simultaneously call <code>writei()</code> on the same inode, modifying the size and block map while <code>readi()</code> is partway through. This produces a torn read — the path string in <code>target[]</code> is a mix of old and new data. The resulting <code>namei(target)</code> call follows a corrupt path.</p>

<p><strong>Bug 3 — iunlockput(ip) called on a never-locked inode:</strong></p>
<pre><code class="language-c">iunlockput(ip);   // called inside the buggy loop
// iunlockput calls iunlock(ip), which calls releasesleep(&ip->lock)
// releasesleep on a lock that was never acquired: undefined behaviour
// On xv6's implementation: panic("releasesleep") or silent memory corruption</code></pre>

<p><strong>Bug 4 — ilock(ip) after the loop — double-lock if loop is skipped:</strong></p>
<pre><code class="language-c">// If ip is NOT a symlink, the while loop is skipped entirely.
// ilock(ip) runs normally — this case works correctly.
// If ip IS a symlink and was followed:
//   The loop ends with the new inode (ip = namei(target)) NOT locked.
//   ilock(ip) below the loop locks it — this case works correctly.
//   But the intermediate iunlockput() was called on an unlocked inode (Bug 3 above).
// So the loop path is broken; the non-loop path works — intermittent failures.</code></pre>

<p><strong>Summary of all bugs:</strong></p>

<table>
  <tr><th>Bug</th><th>Type</th><th>Consequence</th></tr>
  <tr><td>Read ip->type without lock</td><td>Data race</td><td>May see stale, freed, or reused inode type</td></tr>
  <tr><td>readi() without lock</td><td>Data race</td><td>Torn read of target path → corrupt namei() argument</td></tr>
  <tr><td>iunlockput() on unlocked inode</td><td>Lock discipline violation</td><td>Panic or silent corruption of sleep lock state</td></tr>
  <tr><td>Inconsistent lock state on exit</td><td>Lock discipline violation</td><td>Subsequent directory check reads ip->type without lock guarantee</td></tr>
</table>

</div>
</div>
