---
layout: page
title: "Lab Predictions - lab3-r3n8"
lab: lab3
description: "Exam predictions for Lab3-w8: doubly-indirect blocks and symbolic links."
---

## Easy

<div class="var-question">
<div class="var-qnum">Q1 <span class="diff diff-easy">Easy</span></div>

What is the maximum file size in the <strong>original</strong> xv6 file system, and why is it limited?

<button class="answer-toggle">▼ Show Answer</button>
<div class="answer-body">

<p><strong>Original inode layout — 13 slots total:</strong></p>
<pre><code class="language-c">// kernel/fs.h (before lab)
#define NDIRECT   12
#define NINDIRECT (BSIZE / sizeof(uint))  // 1024/4 = 256

struct dinode {
    // ...
    uint addrs[NDIRECT + 1];  // 12 direct + 1 singly-indirect = 13 slots
};</code></pre>

<p><strong>Block count calculation:</strong></p>
<ol>
  <li>Slots 0–11 (Direct): each points to 1 data block → <strong>12 blocks</strong></li>
  <li>Slot 12 (Singly-indirect): points to a map block holding 256 addresses → <strong>256 blocks</strong></li>
  <li>Total: <code>12 + 256 = 268 blocks</code></li>
  <li>Size: <code>268 × 1024 bytes = 268 KB</code></li>
</ol>

<p>Writing block 269 makes <code>bmap()</code> hit the end of <code>addrs[]</code> → returns 0 → write returns 0 → <code>bigfile</code> prints "file is too small".</p>

</div>
</div>

<div class="var-question">
<div class="var-qnum">Q2 <span class="diff diff-easy">Easy</span></div>

After the Lab3 modification, how many total blocks can a file have? Show the full calculation.

<button class="answer-toggle">▼ Show Answer</button>
<div class="answer-body">

<p><strong>New inode layout — still 13 slots, but repurposed:</strong></p>
<pre><code class="language-c">// After lab: NDIRECT = 11
// Slot 0–10:  Direct (11 blocks)
// Slot 11:    Singly-indirect  (1 × 256 = 256 blocks)
// Slot 12:    Doubly-indirect  (256 × 256 = 65,536 blocks)</code></pre>

<p><strong>Calculation:</strong></p>
<ol>
  <li>Direct: <code>11 blocks</code></li>
  <li>Singly-indirect: <code>256 blocks</code></li>
  <li>Doubly-indirect: <code>256 secondary maps × 256 data blocks each = 65,536 blocks</code></li>
  <li><strong>Total: 11 + 256 + 65,536 = 65,803 blocks ≈ 64 MB</strong></li>
</ol>

<p><strong>The trade-off:</strong> We sacrificed 1 direct block (12→11) to gain 65,536 doubly-indirect blocks. That's trading 1 KB of direct access for 64 MB of capacity.</p>

</div>
</div>

<div class="var-question">
<div class="var-qnum">Q3 <span class="diff diff-easy">Easy</span></div>

What is a symbolic link and how does it differ from a regular file?

<button class="answer-toggle">▼ Show Answer</button>
<div class="answer-body">

<p><strong>Regular file vs symbolic link:</strong></p>

<table>
  <tr><th>Property</th><th>Regular file (T_FILE)</th><th>Symlink (T_SYMLINK)</th></tr>
  <tr><td>inode type</td><td><code>T_FILE = 2</code></td><td><code>T_SYMLINK = 4</code></td></tr>
  <tr><td>Data blocks contain</td><td>User data (bytes)</td><td>A path string (e.g. <code>"/usr/bin/sh"</code>)</td></tr>
  <tr><td>open("name")</td><td>Opens the file itself</td><td>Kernel follows the path → opens target</td></tr>
  <tr><td>Target deleted?</td><td>File still exists</td><td>Dangling link → open returns -1</td></tr>
  <tr><td>Hard link count</td><td>Counts references</td><td>nlink = 1 (only the symlink itself)</td></tr>
</table>

<p><strong>How a symlink is stored:</strong></p>
<pre><code class="language-c">// sys_symlink("target", "path") creates:
ip = create(path, T_SYMLINK, 0, 0);   // new inode, type = 4
writei(ip, 0, (uint64)target, 0, strlen(target));
// ip's data block now contains: "/usr/bin/sh\0"</code></pre>

</div>
</div>

<div class="var-question">
<div class="var-qnum">Q4 <span class="diff diff-easy">Easy</span></div>

What does <code>brelse()</code> do and why must it always be called after <code>bread()</code>?

<button class="answer-toggle">▼ Show Answer</button>
<div class="answer-body">

<p><strong>What happens inside bread() and brelse():</strong></p>
<ol>
  <li><code>bread(dev, blockno)</code> → finds or loads the block into the <strong>buffer cache</strong>, marks it <strong>locked</strong>, returns pointer</li>
  <li>You read data from <code>bp-&gt;data[]</code></li>
  <li><code>brelse(bp)</code> → <strong>unlocks</strong> the buffer, moves it to the MRU head (available for reuse)</li>
</ol>

<p><strong>The buffer cache is tiny — what happens without brelse:</strong></p>
<pre><code class="language-c">struct buf *bp1 = bread(dev, block1);  // locks slot 1
struct buf *bp2 = bread(dev, block2);  // locks slot 2
struct buf *bp3 = bread(dev, block3);  // locks slot 3
// ... keep going without brelse ...
struct buf *bpN = bread(dev, blockN);  // ALL slots locked!
// Next bread() call: no free slot → kernel PANICS: "bget: no buffers"</code></pre>

<p><strong>The rule:</strong></p>
<pre><code class="language-c">struct buf *bp = bread(ip->dev, addr);
uint *a = (uint *)bp->data;
uint target_addr = a[index];   // extract what you need
brelse(bp);                    // IMMEDIATELY release — before any other bread()
// Now use target_addr (NOT bp, which is now invalid)</code></pre>

</div>
</div>

---

## Medium

<div class="var-question">
<div class="var-qnum">Q5 <span class="diff diff-med">Medium</span></div>

In <code>bmap()</code>, given logical block number <code>bn = 500</code>, calculate <code>index1</code> and <code>index2</code> for the doubly-indirect section. Show each step.

<button class="answer-toggle">▼ Show Answer</button>
<div class="answer-body">

<p><strong>Step 1 — Subtract direct range:</strong></p>
<pre><code class="language-c">bn = 500
bn -= NDIRECT;   // 500 - 11 = 489
// Is 489 < NINDIRECT (256)? No → not singly-indirect</code></pre>

<p><strong>Step 2 — Subtract singly-indirect range:</strong></p>
<pre><code class="language-c">bn -= NINDIRECT;  // 489 - 256 = 233
// Is 233 < NDINDIRECT (65536)? Yes → doubly-indirect section
// bn is now the LOCAL offset within the doubly-indirect range</code></pre>

<p><strong>Step 3 — Calculate indices:</strong></p>
<pre><code class="language-c">uint index1 = bn / NINDIRECT;  // 233 / 256 = 0  → use secondary map #0
uint index2 = bn % NINDIRECT;  // 233 % 256 = 233 → use slot #233 in that map</code></pre>

<p><strong>What they mean physically:</strong></p>
<ol>
  <li><code>ip-&gt;addrs[12]</code> → Master Map block</li>
  <li>Master Map entry at <code>[index1=0]</code> → Secondary Map block #0</li>
  <li>Secondary Map entry at <code>[index2=233]</code> → the actual <strong>data block</strong> for logical block 500</li>
</ol>

</div>
</div>

<div class="var-question">
<div class="var-qnum">Q6 <span class="diff diff-med">Medium</span></div>

Why must <code>struct inode</code> in <code>file.h</code> and <code>struct dinode</code> in <code>fs.h</code> always have the same number of elements in <code>addrs[]</code>?

<button class="answer-toggle">▼ Show Answer</button>
<div class="answer-body">

<p><strong>What each struct is for:</strong></p>
<ul>
  <li><code>struct dinode</code> (<code>fs.h</code>): the <strong>on-disk</strong> format — physically stored in inode blocks on disk</li>
  <li><code>struct inode</code> (<code>file.h</code>): the <strong>in-memory</strong> copy — what the kernel works with in RAM</li>
</ul>

<p><strong>How they're connected — step by step in ilock():</strong></p>
<ol>
  <li><code>bread()</code> loads the disk block containing the dinode</li>
  <li>Code does: <code>memmove(ip-&gt;addrs, dip-&gt;addrs, sizeof(ip-&gt;addrs))</code></li>
  <li>This copies <code>addrs[]</code> from dinode (disk) to inode (RAM)</li>
  <li>If <code>dinode.addrs[13]</code> but <code>inode.addrs[12]</code>: last entry is <strong>not copied</strong> → doubly-indirect pointer = 0 in RAM → bmap fails to find data blocks</li>
  <li>If reversed: <code>dinode.addrs[12]</code> but <code>inode.addrs[13]</code>: extra slot loaded with garbage → bmap follows a random block number → <strong>data corruption / panic</strong></li>
</ol>

<p><strong>After the lab, both must be:</strong></p>
<pre><code class="language-c">uint addrs[NDIRECT + 2];  // [0..10]=direct, [11]=singly-indirect, [12]=doubly-indirect</code></pre>

</div>
</div>

<div class="var-question">
<div class="var-qnum">Q7 <span class="diff diff-med">Medium</span></div>

Explain <code>O_NOFOLLOW</code>. When would a user program use it, and what happens when opening a symlink without this flag?

<button class="answer-toggle">▼ Show Answer</button>
<div class="answer-body">

<p><strong>How O_NOFOLLOW works in sys_open():</strong></p>
<pre><code class="language-c">// In sys_open(), the symlink-following loop:
while (ip->type == T_SYMLINK && !(omode &amp; O_NOFOLLOW)) {
    // follow the link...
}
// If O_NOFOLLOW is set: condition is false → loop never runs
// → opens the symlink ITSELF, not the target</code></pre>

<p><strong>Without O_NOFOLLOW — what happens step by step:</strong></p>
<ol>
  <li><code>open("link_to_file")</code> → finds symlink inode</li>
  <li><code>ilock(ip)</code> → <code>ip-&gt;type == T_SYMLINK</code> and <code>O_NOFOLLOW</code> not set</li>
  <li>Enter loop: <code>readi()</code> reads <code>"real_file"</code> from symlink's data block</li>
  <li><code>iunlockput(ip)</code> → release symlink inode</li>
  <li><code>namei("real_file")</code> → get real file's inode</li>
  <li>Opens real file transparently — caller never knew it was a symlink</li>
</ol>

<p><strong>Use cases for O_NOFOLLOW:</strong></p>
<ul>
  <li><code>unlink("link_name")</code> — need the symlink's own inode to delete it</li>
  <li>Security checks — verify the link target before following</li>
  <li><code>ls -l</code> — display <code>link_name -&gt; target</code> metadata</li>
  <li><code>readlink()</code> — read what path a link points to</li>
</ul>

</div>
</div>

<div class="var-question">
<div class="var-qnum">Q8 <span class="diff diff-med">Medium</span></div>

Why must the symlink-following loop in <code>sys_open()</code> be placed <strong>after</strong> <code>ilock(ip)</code> but <strong>before</strong> the directory check?

<button class="answer-toggle">▼ Show Answer</button>
<div class="answer-body">

<p><strong>Reason 1 — Must be AFTER ilock(ip):</strong></p>
<ol>
  <li>You need the inode lock to safely read <code>ip-&gt;type</code></li>
  <li>Without the lock: another process could change <code>ip-&gt;type</code> between your read and use (TOCTOU race)</li>
  <li>You need the lock to call <code>readi()</code> — it reads the inode's data blocks, which requires the inode to be stable</li>
</ol>

<pre><code class="language-c">ilock(ip);         // ← MUST acquire lock first
// NOW safe to:
if (ip->type == T_SYMLINK ...)   // read type
    readi(ip, ...)               // read data blocks (target path)</code></pre>

<p><strong>Reason 2 — Must be BEFORE the directory check:</strong></p>
<ol>
  <li>A symlink can point to a <strong>directory</strong> (valid use case: <code>symlink("/data/dir", "/shortcut")</code>)</li>
  <li>The directory check: <code>if(ip-&gt;type == T_DIR &amp;&amp; omode != O_RDONLY)</code></li>
  <li>If you check this on a <strong>symlink</strong> inode: <code>T_SYMLINK != T_DIR</code> → passes incorrectly</li>
  <li>Result: opening a symlink to a directory with <code>O_WRONLY</code> wouldn't be caught</li>
  <li>By resolving the symlink first, the check runs on the <strong>actual target inode</strong> → correct behaviour</li>
</ol>

</div>
</div>

---

## Hard

<div class="var-question">
<div class="var-qnum">Q9 <span class="diff diff-hard">Hard</span></div>

Write the complete doubly-indirect allocation logic for <code>bmap()</code> with all <code>balloc</code>, <code>bread</code>, <code>brelse</code>, and <code>log_write</code> calls. Explain each step.

<button class="answer-toggle">▼ Show Answer</button>
<div class="answer-body">

<pre><code class="language-c">// In bmap(), after singly-indirect section:
bn -= NINDIRECT;

if (bn &lt; NDINDIRECT) {   // NDINDIRECT = 256 * 256 = 65536

    // ── STEP 1: Get or create the Master Map block ──
    // ip->addrs[NDIRECT+1] = slot 12 = doubly-indirect pointer
    if ((addr = ip->addrs[NDIRECT + 1]) == 0) {
        ip->addrs[NDIRECT + 1] = addr = balloc(ip->dev);
        if (addr == 0) return 0;   // disk full
    }

    // ── STEP 2: Read the Master Map block into buffer cache ──
    struct buf *bp1 = bread(ip->dev, addr);
    uint *a1 = (uint *)bp1->data;

    // ── STEP 3: Calculate which secondary map and which slot ──
    uint idx1 = bn / NINDIRECT;   // which of the 256 secondary maps?
    uint idx2 = bn % NINDIRECT;   // which of the 256 slots in that map?

    // ── STEP 4: Get or create the Secondary Map block ──
    if ((addr = a1[idx1]) == 0) {
        a1[idx1] = addr = balloc(ip->dev);
        if (addr == 0) { brelse(bp1); return 0; }
        log_write(bp1);   // mark master map dirty → survives crash
    }

    // ── STEP 5: MUST release bp1 before reading bp2 ──
    // Buffer cache has limited slots — holding bp1 while bread()ing bp2
    // could exhaust the cache and cause a panic
    brelse(bp1);

    // ── STEP 6: Read the Secondary Map block ──
    struct buf *bp2 = bread(ip->dev, addr);
    uint *a2 = (uint *)bp2->data;

    // ── STEP 7: Get or create the actual Data block ──
    if ((addr = a2[idx2]) == 0) {
        a2[idx2] = addr = balloc(ip->dev);
        if (addr == 0) { brelse(bp2); return 0; }
        log_write(bp2);   // mark secondary map dirty
    }

    // ── STEP 8: Release and return ──
    brelse(bp2);
    return addr;   // physical block number of the data block
}
panic("bmap: out of range");</code></pre>

<p><strong>Critical rule:</strong> Always call <code>brelse(bp1)</code> before <code>bread(bp2)</code> — never hold two buffer locks simultaneously in a single code path.</p>

</div>
</div>

<div class="var-question">
<div class="var-qnum">Q10 <span class="diff diff-hard">Hard</span></div>

Write the doubly-indirect cleanup in <code>itrunc()</code>. What happens if you forget to <code>bfree</code> the Master Map block itself?

<button class="answer-toggle">▼ Show Answer</button>
<div class="answer-body">

<p><strong>Complete doubly-indirect itrunc() code:</strong></p>
<pre><code class="language-c">// Add AFTER the singly-indirect section in itrunc():

if (ip->addrs[NDIRECT + 1]) {

    // Step 1: Read the Master Map
    struct buf *bp1 = bread(ip->dev, ip->addrs[NDIRECT + 1]);
    uint *a1 = (uint *)bp1->data;

    // Step 2: For each Secondary Map entry in the Master Map
    for (int i = 0; i &lt; NINDIRECT; i++) {
        if (a1[i]) {

            // Step 3: Read each Secondary Map
            struct buf *bp2 = bread(ip->dev, a1[i]);
            uint *a2 = (uint *)bp2->data;

            // Step 4: Free all data blocks in this Secondary Map
            for (int j = 0; j &lt; NINDIRECT; j++) {
                if (a2[j])
                    bfree(ip->dev, a2[j]);   // free actual data block
            }

            // Step 5: Release and free the Secondary Map block itself
            brelse(bp2);
            bfree(ip->dev, a1[i]);           // ← free secondary map block
        }
    }

    // Step 6: Release and free the Master Map block itself
    brelse(bp1);
    bfree(ip->dev, ip->addrs[NDIRECT + 1]);  // ← free master map block

    // Step 7: Clear the inode slot
    ip->addrs[NDIRECT + 1] = 0;
}</code></pre>

<p><strong>What happens if you forget <code>bfree(ip-&gt;dev, ip-&gt;addrs[NDIRECT+1])</code>:</strong></p>
<ol>
  <li>The Master Map block (1 block per large file) is never returned to the free list</li>
  <li>Its bit in the bitmap stays set (= "allocated") even though no inode references it</li>
  <li>Each time you create and delete a large file: <strong>1 block leaked permanently</strong></li>
  <li>After many such operations: free block count drops to 0 even though most disk is "free"</li>
  <li>Next <code>balloc()</code> call: scans entire bitmap, finds no free bit → <code>panic("balloc: out of blocks")</code></li>
</ol>

</div>
</div>

<div class="var-question">
<div class="var-qnum">Q11 <span class="diff diff-hard">Hard</span></div>

Symlink chain: A → B → A (circular). Without a depth limit, what happens? Implement the correct loop guard with proper <code>ilock</code>/<code>iunlockput</code> usage.

<button class="answer-toggle">▼ Show Answer</button>
<div class="answer-body">

<p><strong>Without depth limit — step-by-step infinite loop:</strong></p>
<ol>
  <li><code>namei("A")</code> → A's inode; <code>ilock(ip)</code>; type = T_SYMLINK</li>
  <li><code>readi()</code> → target = <code>"B"</code>; <code>iunlockput(ip)</code></li>
  <li><code>namei("B")</code> → B's inode; <code>ilock(ip)</code>; type = T_SYMLINK</li>
  <li><code>readi()</code> → target = <code>"A"</code>; <code>iunlockput(ip)</code></li>
  <li>Back to step 1 → <strong>infinite loop in kernel space</strong></li>
  <li>Kernel stack grows with each iteration → <strong>stack overflow → panic</strong></li>
  <li>Process can never be interrupted out of this loop</li>
</ol>

<p><strong>Correct implementation with depth guard:</strong></p>
<pre><code class="language-c">// In sys_open(), AFTER ilock(ip), BEFORE directory check:
#define MAX_SYMLINK_DEPTH 10

int depth = 0;
while (ip->type == T_SYMLINK &amp;&amp; !(omode &amp; O_NOFOLLOW)) {

    // Step 1: Check depth BEFORE doing anything
    if (depth++ >= MAX_SYMLINK_DEPTH) {
        iunlockput(ip);   // release before returning!
        end_op();
        return -1;        // ELOOP: too many levels of symbolic links
    }

    // Step 2: Read target path from symlink's data block
    char target[MAXPATH];
    int n = readi(ip, 0, (uint64)target, 0, MAXPATH - 1);
    if (n &lt;= 0) { iunlockput(ip); end_op(); return -1; }
    target[n] = '\0';   // null-terminate! readi does NOT add \0

    // Step 3: Release OLD inode BEFORE looking up new one
    // CRITICAL: never hold two inode locks simultaneously (deadlock risk)
    iunlockput(ip);

    // Step 4: Look up the target
    if ((ip = namei(target)) == 0) {
        end_op();
        return -1;   // dangling symlink — target doesn't exist
    }

    // Step 5: Lock the NEW inode and loop
    ilock(ip);
    // → loop checks ip->type again → if T_FILE, exits loop, proceeds normally
}</code></pre>

<p><strong>Key subtlety:</strong> <code>iunlockput(ip)</code> must come BEFORE <code>namei(target)</code>. If A and B simultaneously open the circular chain in opposite directions (A→B vs B→A), holding A's lock while requesting B's lock (and vice versa) is a <strong>classic deadlock</strong>.</p>

</div>
</div>
