---
layout: page
title: "Lab Predictions - lab3-g7h8"
lab: lab3
description: "Exam predictions for Lab3-w8: code completion questions, make clean necessity, on-disk format changes, and symlink chain mechanics."
---

## Easy

<div class="var-question">
<div class="var-qnum">Q1 <span class="diff diff-easy">Easy</span></div>

Fill in the blank: In <code>bmap()</code>, after handling the singly-indirect case, the code does <code>bn -= NINDIRECT</code> before entering the doubly-indirect block. Why is this subtraction necessary, and what value does <code>bn</code> represent after it?

<div class="answer-content">

<p><strong>Full subtraction sequence in bmap():</strong></p>
<pre><code class="language-c">static uint bmap(struct inode *ip, uint bn) {

    if (bn < NDIRECT) { ... return addr; }
    bn -= NDIRECT;          // subtract 11: bn now relative to singly-indirect start

    if (bn < NINDIRECT) { ... return addr; }
    bn -= NINDIRECT;        // subtract 256: bn now relative to doubly-indirect start

    if (bn < NDINDIRECT) {
        // Now bn is the LOCAL offset within the doubly-indirect region
        uint idx1 = bn / NINDIRECT;   // which secondary map (0–255)?
        uint idx2 = bn % NINDIRECT;   // which data block in that map (0–255)?
        ...
    }
    panic("bmap: out of range");
}</code></pre>

<p><strong>What bn represents after each subtraction:</strong></p>
<table>
  <tr><th>After</th><th>bn represents</th><th>Example (original bn=523)</th></tr>
  <tr><td>Initial bn</td><td>Logical file block number</td><td>523</td></tr>
  <tr><td>bn -= NDIRECT (11)</td><td>Offset from start of singly-indirect region</td><td>512</td></tr>
  <tr><td>bn -= NINDIRECT (256)</td><td>Offset from start of doubly-indirect region</td><td>256</td></tr>
  <tr><td>idx1 = bn / 256</td><td>Which secondary map to use</td><td>1 (secondary map #1)</td></tr>
  <tr><td>idx2 = bn % 256</td><td>Which slot within that secondary map</td><td>0 (first slot)</td></tr>
</table>

<p><strong>Why the subtraction is necessary:</strong></p>
<p>The division and modulo only make sense if <code>bn</code> is the offset within the doubly-indirect region (0 to 65535), not the absolute logical block number. Without the subtractions, <code>bn=523 / 256 = 2</code> — pointing to secondary map #2 — but block 523 actually lives in secondary map #1. The subtractions normalise <code>bn</code> to the correct local offset.</p>

</div>
</div>

<div class="var-question">
<div class="var-qnum">Q2 <span class="diff diff-easy">Easy</span></div>

What is a "dangling symlink" and how does <code>sys_open()</code> handle it? Write the code path that handles this case.

<div class="answer-content">

<p><strong>What a dangling symlink is:</strong></p>
<p>A symlink whose target no longer exists. Example: you create <code>mylink → /tmp/file.txt</code>, then delete <code>/tmp/file.txt</code>. The symlink inode still exists with the path string <code>"/tmp/file.txt"</code> stored in it, but the target is gone.</p>

<p><strong>How sys_open() handles it:</strong></p>
<pre><code class="language-c">while (ip->type == T_SYMLINK && !(omode & O_NOFOLLOW)) {
    if (depth >= 10) { iunlockput(ip); end_op(); return -1; }
    depth++;

    char target[MAXPATH];
    int n = readi(ip, 0, (uint64)target, 0, MAXPATH - 1);
    if (n <= 0) { iunlockput(ip); end_op(); return -1; }
    target[n] = '\0';

    iunlockput(ip);        // release the symlink inode

    ip = namei(target);    // try to find the target
    if (ip == 0) {
        end_op();
        return -1;         // ← DANGLING SYMLINK: namei returns 0 (not found)
    }
    ilock(ip);
}</code></pre>

<p><strong>What the user sees:</strong></p>
<pre><code class="language-c">// In user space:
fd = open("mylink", O_RDONLY);
// fd = -1  (sys_open returned -1 for dangling symlink)
// On Linux this sets errno = ENOENT (No such file or directory)
// xv6 has no errno, but the return value -1 signals failure</code></pre>

<p><strong>Note:</strong> The symlink file itself still exists — it just can't be opened through following. You can still open the symlink itself with <code>O_NOFOLLOW</code> to see what path it stores.</p>

</div>
</div>

<div class="var-question">
<div class="var-qnum">Q3 <span class="diff diff-easy">Easy</span></div>

Why does <code>kernel/fs.h</code> define <code>NDINDIRECT = NINDIRECT * NINDIRECT</code> rather than just writing <code>256 * 256</code> or <code>65536</code> as a literal?

<div class="answer-content">

<p><strong>The definition:</strong></p>
<pre><code class="language-c">#define NINDIRECT   (BSIZE / sizeof(uint))      // = 1024/4 = 256
#define NDINDIRECT  (NINDIRECT * NINDIRECT)     // = 256*256 = 65536</code></pre>

<p><strong>Why use the symbolic form instead of a literal:</strong></p>

<table>
  <tr><th>Reason</th><th>Explanation</th></tr>
  <tr><td><strong>DRY principle</strong></td><td>If BSIZE ever changes (e.g., to 4096 for a real filesystem), NINDIRECT automatically becomes 1024, and NDINDIRECT becomes 1024×1024 = 1,048,576 — without any manual updates</td></tr>
  <tr><td><strong>Self-documenting</strong></td><td><code>NINDIRECT * NINDIRECT</code> communicates the concept "256 secondary maps × 256 entries each". A literal 65536 hides this meaning.</td></tr>
  <tr><td><strong>Consistency</strong></td><td>The bmap() code uses <code>bn / NINDIRECT</code> and <code>bn % NINDIRECT</code>. Using NINDIRECT throughout ensures the index calculation is always consistent with the actual capacity.</td></tr>
  <tr><td><strong>Correctness check</strong></td><td>The compiler can verify that NDINDIRECT = NINDIRECT² at compile time. If you use literals and make an arithmetic error (e.g., write 65803 instead of 65536 for NDINDIRECT), the check in bmap would be wrong.</td></tr>
</table>

<p><strong>MAXFILE calculation using symbolic constants:</strong></p>
<pre><code class="language-c">#define MAXFILE (NDIRECT + NINDIRECT + NDINDIRECT)
// = 11 + 256 + 65536 = 65803
// If BSIZE changes to 4096: MAXFILE = 11 + 1024 + 1048576 = 1049611 blocks</code></pre>

</div>
</div>

<div class="var-question">
<div class="var-qnum">Q4 <span class="diff diff-easy">Easy</span></div>

The lab says "Do not forget to brelse() each block that you bread()." Give three distinct negative consequences of forgetting a single brelse() call in bmap().

<div class="answer-content">

<p><strong>Consequence 1 — Buffer cache exhaustion (most immediate):</strong></p>
<p>The buffer cache has only NBUF=30 slots. Each missing <code>brelse()</code> permanently uses one slot. After 30 such operations, <code>bget()</code> finds no free buffer:</p>
<pre><code class="language-c">// bget() after all slots are refcnt > 0:
panic("bget: no buffers");
// Kernel crashes — entire system dies</code></pre>

<p><strong>Consequence 2 — LRU list corruption (no eviction):</strong></p>
<p>The buffer that isn't released stays permanently at <code>refcnt > 0</code>. It can never be selected for LRU eviction. Over time, the effective cache size shrinks, reducing hit rates and increasing disk reads for all processes.</p>

<p><strong>Consequence 3 — Memory waste from pinned dirty buffers:</strong></p>
<p>If the leaked buffer was modified (via <code>log_write()</code>), it stays pinned in the cache with <code>B_DIRTY</code>. The log layer calls <code>bpin()</code> to keep it during a transaction. If <code>brelse()</code> is never called, the buffer's refcnt never drops to 0, and the log layer cannot unpin it via <code>bunpin()</code>. This can cause the log to fill up, causing <code>begin_op()</code> to sleep forever waiting for log space.</p>

<p><strong>Practical debugging tip:</strong></p>
<p>If your bigfile test panics with "bget: no buffers" after writing ~30–60 blocks, the most likely cause is a missing <code>brelse()</code> in your doubly-indirect <code>bmap()</code> code. Count your <code>bread()</code> calls and verify each has exactly one matching <code>brelse()</code>.</p>

</div>
</div>

---

## Medium

<div class="var-question">
<div class="var-qnum">Q5 <span class="diff diff-med">Medium</span></div>

Complete the following incomplete <code>itrunc()</code> doubly-indirect section. Five lines are missing — identify each blank and write the correct code.

<div class="answer-content">

<p><strong>The incomplete code (blanks marked with ?):</strong></p>
<pre><code class="language-c">if (ip->addrs[NDIRECT + 1]) {
    struct buf *bp1 = bread(ip->dev, ______?______);   // blank 1
    uint *a1 = (uint *)bp1->data;

    for (int i = 0; i < NINDIRECT; i++) {
        if (a1[i]) {
            struct buf *bp2 = bread(ip->dev, ______?______);  // blank 2
            uint *a2 = (uint *)bp2->data;

            for (int j = 0; j < NINDIRECT; j++) {
                if (a2[j])
                    bfree(______?______, a2[j]);   // blank 3
            }
            brelse(bp2);
            bfree(______?______, a1[i]);           // blank 4
        }
    }
    brelse(bp1);
    bfree(______?______, ip->addrs[NDIRECT + 1]);  // blank 5
    ip->addrs[NDIRECT + 1] = 0;
}</code></pre>

<p><strong>Answers:</strong></p>
<table>
  <tr><th>Blank</th><th>Code</th><th>Reason</th></tr>
  <tr><td>1</td><td><code>ip->addrs[NDIRECT + 1]</code></td><td>The master map block address is stored in slot 12 (NDIRECT+1) of the inode's addrs array</td></tr>
  <tr><td>2</td><td><code>a1[i]</code></td><td><code>a1[i]</code> is the disk block number of secondary map #i, read from the master map</td></tr>
  <tr><td>3</td><td><code>ip->dev</code></td><td>bfree needs the device number to find the correct bitmap; ip->dev is the device this inode lives on</td></tr>
  <tr><td>4</td><td><code>ip->dev</code></td><td>Same reason — freeing the secondary map block itself from the same device</td></tr>
  <tr><td>5</td><td><code>ip->dev</code></td><td>Same reason — freeing the master map block</td></tr>
</table>

<p><strong>Complete correct code:</strong></p>
<pre><code class="language-c">if (ip->addrs[NDIRECT + 1]) {
    struct buf *bp1 = bread(ip->dev, ip->addrs[NDIRECT + 1]);
    uint *a1 = (uint *)bp1->data;

    for (int i = 0; i < NINDIRECT; i++) {
        if (a1[i]) {
            struct buf *bp2 = bread(ip->dev, a1[i]);
            uint *a2 = (uint *)bp2->data;

            for (int j = 0; j < NINDIRECT; j++) {
                if (a2[j])
                    bfree(ip->dev, a2[j]);    // free each data block
            }
            brelse(bp2);
            bfree(ip->dev, a1[i]);             // free the secondary map block
        }
    }
    brelse(bp1);
    bfree(ip->dev, ip->addrs[NDIRECT + 1]);   // free the master map block
    ip->addrs[NDIRECT + 1] = 0;               // clear the inode slot
}</code></pre>

</div>
</div>

<div class="var-question">
<div class="var-qnum">Q6 <span class="diff diff-med">Medium</span></div>

A student runs bigfile, it writes 6580 blocks successfully. They then run bigfile a second time in the same xv6 session without rebooting. The second run writes only 268 blocks. Assuming their bmap() is correct, what is wrong with their itrunc() implementation?

<div class="answer-content">

<p><strong>Why bigfile fails on the second run:</strong></p>
<p>When bigfile runs the second time, it opens <code>"big.file"</code> with <code>O_CREATE</code>. Since the file already exists, the kernel calls <code>itrunc(ip)</code> to reset it to zero length before writing new content.</p>

<p>If <code>itrunc()</code> doesn't free the doubly-indirect blocks:</p>
<ol>
  <li>The doubly-indirect data blocks (267–6579) remain marked as allocated in the free-space bitmap</li>
  <li>The singly-indirect region (blocks 11–266) is freed correctly</li>
  <li>When bigfile writes again, <code>bmap()</code> tries to allocate new doubly-indirect blocks via <code>balloc()</code></li>
  <li>But the bitmap still shows those blocks as used → <code>balloc()</code> finds only the small number of blocks that were freed (11 direct + 1 singly-indirect map + 256 singly-indirect data = 268 blocks)</li>
  <li>After writing 268 blocks, balloc() returns 0 (no free blocks) → write() returns 0 → bigfile stops at 268 blocks</li>
</ol>

<p><strong>Diagnosis checklist for itrunc():</strong></p>
<pre><code class="language-c">// Check each of these is present:

// 1. Is the doubly-indirect block checked?
if (ip->addrs[NDIRECT + 1]) {    // ← must be NDIRECT+1, not NDIRECT

// 2. Is bfree called on EACH secondary map?
    bfree(ip->dev, a1[i]);       // ← missing = secondary map leak

// 3. Is bfree called on EACH data block?
    bfree(ip->dev, a2[j]);       // ← missing = data block leak

// 4. Is bfree called on the master map?
bfree(ip->dev, ip->addrs[NDIRECT + 1]);  // ← missing = master map leak

// 5. Is the inode slot cleared?
ip->addrs[NDIRECT + 1] = 0;     // ← missing = next bmap uses stale pointer</code></pre>

<p><strong>The pattern: writes X blocks first time, 268 second time</strong> almost always means itrunc() freed the direct and singly-indirect blocks (268 total) but leaked the doubly-indirect section. The second run can only write as many blocks as were freed.</p>

</div>
</div>

<div class="var-question">
<div class="var-qnum">Q7 <span class="diff diff-med">Medium</span></div>

The lab says to use <code>argstr(0, target, MAXPATH)</code> and <code>argstr(1, path, MAXPATH)</code> in <code>sys_symlink()</code>. Why can't the kernel directly dereference a pointer passed from user space? Explain the memory model difference.

<div class="answer-content">

<p><strong>Why direct pointer dereference fails:</strong></p>
<p>User programs and the kernel run with <strong>different page tables</strong>. When a user program calls <code>symlink("target", "path")</code>, the C calling convention places pointers to the strings in registers <code>a0</code> and <code>a1</code>. These are <em>virtual addresses</em> in the user process's address space.</p>

<pre><code class="language-c">// User space:
// virtual address 0x00001234 → physical page containing "target\0"
// (mapped in user page table)

// In kernel, after syscall:
// The kernel has its OWN page table
// Virtual address 0x00001234 → DIFFERENT physical page (or unmapped!)
// Direct dereference: *(char*)0x00001234 = WRONG memory or page fault</code></pre>

<p><strong>What argstr() does:</strong></p>
<pre><code class="language-c">int argstr(int n, char *buf, int max) {
    uint64 addr;
    argaddr(n, &addr);          // reads the pointer value from the trapframe register
    return fetchstr(addr, buf, max);  // safely copies string from USER address space
}

// fetchstr() uses copyin() which:
// 1. Looks up the physical address of 'addr' in proc->pagetable (user's page table)
// 2. Copies bytes from that physical address into kernel buffer 'buf'
// 3. Stops at '\0' or max bytes
// This is safe because it explicitly uses the user's page table</code></pre>

<p><strong>Address space layout:</strong></p>
<table>
  <tr><th>Region</th><th>Accessible to</th><th>Notes</th></tr>
  <tr><td>0x0 – MAXVA/2</td><td>User process</td><td>User code, data, stack, heap</td></tr>
  <tr><td>MAXVA/2 – MAXVA</td><td>Kernel</td><td>Kernel code, data, device mappings</td></tr>
  <tr><td>User addresses in kernel</td><td>Not directly</td><td>Must use copyin()/copyout() to cross the boundary</td></tr>
</table>

<p>This separation is a fundamental OS security mechanism — without it, a malicious user program could pass a kernel address as the "path" and cause the kernel to read or overwrite kernel data.</p>

</div>
</div>

<div class="var-question">
<div class="var-qnum">Q8 <span class="diff diff-med">Medium</span></div>

Compare the <strong>before</strong> and <strong>after</strong> layouts of the xv6 inode's <code>addrs[]</code> array. For a file that occupies exactly 300 blocks, draw which slots in <code>addrs[]</code> are populated in the new layout and what each populated slot contains.

<div class="answer-content">

<p><strong>New layout for a 300-block file:</strong></p>

<table>
  <tr><th>Slot index</th><th>Field name</th><th>Contains (for 300-block file)</th></tr>
  <tr><td>addrs[0]</td><td>Direct block 0</td><td>Physical block # for logical block 0</td></tr>
  <tr><td>addrs[1]</td><td>Direct block 1</td><td>Physical block # for logical block 1</td></tr>
  <tr><td>...</td><td>...</td><td>...</td></tr>
  <tr><td>addrs[10]</td><td>Direct block 10</td><td>Physical block # for logical block 10</td></tr>
  <tr><td>addrs[11]</td><td>Singly-indirect map</td><td>Physical block # of the map block (which contains addresses for blocks 11–266)</td></tr>
  <tr><td>addrs[12]</td><td>Doubly-indirect master map</td><td>Physical block # of the master map (only 1 secondary map needed for blocks 267–299)</td></tr>
</table>

<p><strong>What the master map contains for exactly 300 blocks:</strong></p>
<pre><code class="language-c">// 300 blocks total:
// Blocks 0-10:   direct (11 blocks)
// Blocks 11-266: singly-indirect (256 blocks)
// Blocks 267-299: doubly-indirect (33 blocks)

// Doubly-indirect blocks: 267..299 → 33 blocks
// After subtracting: bn = 0..32
// idx1 = 0/256 = 0 for ALL of them → only secondary map #0 needed

// Master map (addrs[12] → block X):
// a[0] = address of secondary map #0      ← POPULATED
// a[1..255] = 0                            ← EMPTY (never allocated)

// Secondary map #0 (at block X, a[0]):
// a[0..32] = 33 data block addresses      ← POPULATED (blocks 267-299)
// a[33..255] = 0                           ← EMPTY</code></pre>

<p><strong>Visual:</strong></p>
<pre><code>addrs[12] → [Master Map: a[0]=SecMap0, a[1..255]=0]
                               |
                               ↓
                        [SecMap 0: a[0..32]=DataBlocks, a[33..255]=0]
                               |
                        [DataBlock 267] [DataBlock 268] ... [DataBlock 299]</code></pre>

<p><strong>Key point:</strong> For a 300-block file, the master map block IS allocated (since any doubly-indirect blocks exist), but only 1 of its 256 secondary map entries is populated. The other 255 secondary maps are never allocated until needed — lazy allocation.</p>

</div>
</div>

---

## Hard

<div class="var-question">
<div class="var-qnum">Q9 <span class="diff diff-hard">Hard</span></div>

The xv6 write-ahead log uses <code>begin_op()</code> and <code>end_op()</code>. In <code>sys_symlink()</code>, all disk writes (inode creation, directory update, target path write) are wrapped in a single transaction. What would happen if each write used its own separate transaction? Give a specific crash scenario.

<div class="answer-content">

<p><strong>Scenario: three separate transactions</strong></p>
<pre><code class="language-c">// WRONG — three separate transactions:

// Transaction 1: create inode
begin_op();
ip = create(path, T_SYMLINK, 0, 0);
iunlockput(ip);
end_op();   // ← commits: inode allocated on disk, directory entry written

// *** CRASH HERE ***

// Transaction 2: write target path (never reaches this)
begin_op();
writei(ip, 0, (uint64)target, 0, strlen(target));
end_op();

// Transaction 3: never reached either</code></pre>

<p><strong>What the crash leaves on disk:</strong></p>
<ol>
  <li>Transaction 1 committed: inode #24 exists on disk with type=T_SYMLINK, nlink=1, size=0</li>
  <li>Directory entry <code>("mylink", 24)</code> exists in the parent directory</li>
  <li>BUT: the inode's data blocks are empty — size=0, addrs[0]=0</li>
</ol>

<p><strong>Effect on the filesystem after reboot:</strong></p>
<pre><code class="language-c">// User tries to open "mylink":
fd = open("mylink", O_RDONLY);
// sys_open() → ilock(ip#24) → ip->type = T_SYMLINK
// symlink loop: readi(ip, 0, target, 0, MAXPATH) with ip->size=0
// readi returns 0 (nothing to read — n <= 0 check)
// iunlockput(ip); end_op(); return -1;
// fd = -1

// ls shows:
// mylink  T_SYMLINK  24  0  (0-byte symlink — broken, unreachable target)</code></pre>

<p><strong>The orphan problem:</strong></p>
<p>The inode has <code>nlink=1</code> (directory entry exists) so it will never be garbage collected by <code>iput()</code>. The 0-byte T_SYMLINK file exists permanently on disk, wasting an inode slot and the directory entry. There's no way for users to fix this without running fsck or reformatting.</p>

<p><strong>Why a single transaction prevents this:</strong></p>
<p>All three writes (inode creation, directory entry, target path data) commit atomically. On reboot, either all are present or none are. There is no intermediate state where the inode exists but is empty.</p>

</div>
</div>

<div class="var-question">
<div class="var-qnum">Q10 <span class="diff diff-hard">Hard</span></div>

The symlinktest "Chain Test" creates a 4-level chain: <code>link4 → link3 → link2 → link1 → file</code>. Trace the entire execution of <code>open("link4", O_RDONLY)</code> through the symlink-following loop in <code>sys_open()</code>, showing the value of <code>depth</code>, <code>ip</code>, and <code>target</code> at each iteration.

<div class="answer-content">

<p><strong>Setup:</strong></p>
<pre><code class="language-c">// file exists, inode #5
// link1 → "file"   inode #6, data = "file\0"
// link2 → "link1"  inode #7, data = "link1\0"
// link3 → "link2"  inode #8, data = "link2\0"
// link4 → "link3"  inode #9, data = "link3\0"</code></pre>

<p><strong>Before loop:</strong></p>
<pre><code class="language-c">ip = namei("link4");   // ip = inode #9 (T_SYMLINK)
ilock(ip);
depth = 0;</code></pre>

<p><strong>Iteration 1:</strong></p>
<table>
  <tr><th>Step</th><th>Value</th></tr>
  <tr><td>Condition check</td><td>ip->type == T_SYMLINK ✓, depth(0) >= 10? NO</td></tr>
  <tr><td>depth++</td><td>depth = 1</td></tr>
  <tr><td>readi(ip#9)</td><td>target = "link3\0"</td></tr>
  <tr><td>iunlockput(ip#9)</td><td>inode #9 released</td></tr>
  <tr><td>ip = namei("link3")</td><td>ip = inode #8 (T_SYMLINK)</td></tr>
  <tr><td>ilock(ip#8)</td><td>inode #8 locked</td></tr>
</table>

<p><strong>Iteration 2:</strong></p>
<table>
  <tr><th>Step</th><th>Value</th></tr>
  <tr><td>Condition check</td><td>ip->type == T_SYMLINK ✓, depth(1) >= 10? NO</td></tr>
  <tr><td>depth++</td><td>depth = 2</td></tr>
  <tr><td>readi(ip#8)</td><td>target = "link2\0"</td></tr>
  <tr><td>iunlockput(ip#8)</td><td>inode #8 released</td></tr>
  <tr><td>ip = namei("link2")</td><td>ip = inode #7 (T_SYMLINK)</td></tr>
  <tr><td>ilock(ip#7)</td><td>inode #7 locked</td></tr>
</table>

<p><strong>Iteration 3:</strong></p>
<table>
  <tr><th>Step</th><th>Value</th></tr>
  <tr><td>Condition check</td><td>ip->type == T_SYMLINK ✓, depth(2) >= 10? NO</td></tr>
  <tr><td>depth++</td><td>depth = 3</td></tr>
  <tr><td>readi(ip#7)</td><td>target = "link1\0"</td></tr>
  <tr><td>iunlockput(ip#7)</td><td>inode #7 released</td></tr>
  <tr><td>ip = namei("link1")</td><td>ip = inode #6 (T_SYMLINK)</td></tr>
  <tr><td>ilock(ip#6)</td><td>inode #6 locked</td></tr>
</table>

<p><strong>Iteration 4:</strong></p>
<table>
  <tr><th>Step</th><th>Value</th></tr>
  <tr><td>Condition check</td><td>ip->type == T_SYMLINK ✓, depth(3) >= 10? NO</td></tr>
  <tr><td>depth++</td><td>depth = 4</td></tr>
  <tr><td>readi(ip#6)</td><td>target = "file\0"</td></tr>
  <tr><td>iunlockput(ip#6)</td><td>inode #6 released</td></tr>
  <tr><td>ip = namei("file")</td><td>ip = inode #5 (T_FILE)</td></tr>
  <tr><td>ilock(ip#5)</td><td>inode #5 locked</td></tr>
</table>

<p><strong>Loop exit:</strong></p>
<pre><code class="language-c">// Condition check: ip->type == T_FILE (not T_SYMLINK) → loop exits
// Continue with normal open on inode #5
// filealloc() → fdalloc() → return fd

// Final state: depth=4, ip=inode#5 (T_FILE), locked
// User receives a valid fd pointing to "file"</code></pre>

</div>
</div>

<div class="var-question">
<div class="var-qnum">Q11 <span class="diff diff-hard">Hard</span></div>

Explain why the xv6 implementation stores the symlink target as raw bytes in the inode's data blocks (using <code>writei()</code>) rather than as a field in the inode struct itself. What are the size implications of each approach, and why does the data-block approach scale better?

<div class="answer-content">

<p><strong>Approach 1: Target as a field in the inode struct</strong></p>
<pre><code class="language-c">// Hypothetical struct dinode with embedded target:
struct dinode {
    short type;
    short major;
    short minor;
    short nlink;
    uint  size;
    uint  addrs[NDIRECT + 2];   // 52 bytes
    char  symlink_target[256];  // ← embedded target path
};
// sizeof: 6 + 4 + 52 + 256 = 318 bytes
// Must fit in a power of 2 for IPB (inodes per block) to work nicely
// 318 bytes: IPB = 1024/318 = 3.2 → only 3 inodes per block, lots of wasted space</code></pre>

<p><strong>Approach 2: Target in data blocks (actual implementation)</strong></p>
<pre><code class="language-c">// struct dinode stays small:
struct dinode {
    short type;
    short major;
    short minor;
    short nlink;
    uint  size;
    uint  addrs[NDIRECT + 2];
};
// sizeof = 6 + 4 + 52 = 62 bytes → pad to 64 bytes
// IPB = 1024/64 = 16 inodes per block (efficient!)

// Target path stored in first data block:
// Allocated via bmap(ip, 0), written via writei()
// Can hold up to BSIZE = 1024 bytes (MAXPATH = 128 in xv6)</code></pre>

<p><strong>Size comparison:</strong></p>
<table>
  <tr><th>Approach</th><th>Inode size</th><th>Inodes/block</th><th>Max path length</th></tr>
  <tr><td>Embedded field (256 bytes)</td><td>~318 bytes</td><td>3</td><td>256 chars</td></tr>
  <tr><td>Embedded field (128 bytes)</td><td>~190 bytes</td><td>5</td><td>128 chars</td></tr>
  <tr><td>Data blocks (actual)</td><td>64 bytes</td><td>16</td><td>BSIZE = 1024 chars</td></tr>
</table>

<p><strong>Why data blocks scale better:</strong></p>
<ol>
  <li><strong>Packing efficiency:</strong> 16 inodes per block vs 3–5 with embedded paths. Fewer inode blocks needed for the same number of files.</li>
  <li><strong>Uniform handling:</strong> <code>writei()</code> and <code>readi()</code> already exist for all file types. Symlink creation/reading reuses the same code as regular files — no special cases needed in the I/O path.</li>
  <li><strong>Variable-length paths:</strong> A short path like "a" uses only 1 byte of data block space. An embedded field would waste the rest of the 128/256 byte allocation regardless.</li>
  <li><strong>No struct layout change:</strong> Adding a path field changes the inode size and breaks the on-disk format. Using data blocks requires only a new <code>type</code> value (T_SYMLINK=4) — the struct layout is unchanged.</li>
</ol>

</div>
</div>
