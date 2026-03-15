---
layout: page
title: "Lab Predictions - lab3-c8r1"
lab: lab3
description: "Exam predictions for Lab3-w8: buffer cache contract, bread/brelse, balloc, bitmap internals, and mkfs output."
---

## Easy

<div class="var-question">
<div class="var-qnum">Q1 <span class="diff diff-easy">Easy</span></div>

What does <code>bread(dev, blockno)</code> return and what obligations does the caller take on?

<div class="answer-content">

<p><strong>What bread() returns:</strong></p>
<pre><code class="language-c">struct buf *bp = bread(ip->dev, addr);
// bp points to a buffer cache slot containing the disk block's data.
// bp->data is a 1024-byte array — the raw bytes from disk block 'addr'.
// The buffer is LOCKED — no other kernel code can access it until brelse() is called.</code></pre>

<p><strong>The three obligations bread() imposes on the caller:</strong></p>

<table>
  <tr><th>Obligation</th><th>What happens if violated</th></tr>
  <tr><td>Call <code>brelse(bp)</code> when done</td><td>The buffer slot stays locked forever — after NBUF=30 such leaks the next <code>bread()</code> call panics: <code>"bget: no buffers"</code></td></tr>
  <tr><td>Do not access <code>bp->data</code> after <code>brelse()</code></td><td>The buffer may be immediately reused for a different disk block — reading it returns another file's data</td></tr>
  <tr><td>Call <code>log_write(bp)</code> before <code>brelse()</code> if you modified <code>bp->data</code></td><td>The modification is lost on crash — the buffer cache may write it back eventually, but there is no crash safety guarantee</td></tr>
</table>

<p><strong>The canonical usage pattern:</strong></p>
<pre><code class="language-c">struct buf *bp = bread(ip->dev, blockno);   // acquire
uint *a = (uint*)bp->data;                  // use
a[index] = new_value;                       // modify
log_write(bp);                              // mark dirty (if modified)
brelse(bp);                                 // release — bp is now invalid
// DO NOT use bp or a after this line</code></pre>

</div>
</div>

<div class="var-question">
<div class="var-qnum">Q2 <span class="diff diff-easy">Easy</span></div>

What does <code>balloc(dev)</code> do step by step, and what does it return when the disk is full?

<div class="answer-content">

<p><strong>What balloc() does:</strong></p>
<pre><code class="language-c">// kernel/fs.c — simplified balloc() behaviour:
uint balloc(uint dev)
{
    for(b = 0; b &lt; sb.size; b += BPB){         // iterate over bitmap blocks
        bp = bread(dev, BBLOCK(b, sb));          // load bitmap block
        for(bi = 0; bi &lt; BPB &amp;&amp; b+bi &lt; sb.size; bi++){
            m = 1 &lt;&lt; (bi % 8);
            if((bp->data[bi/8] &amp; m) == 0){      // bit = 0 → block is free
                bp->data[bi/8] |= m;             // set bit = 1 → mark allocated
                log_write(bp);
                brelse(bp);
                bzero(dev, b+bi);                // zero the newly allocated block
                return b+bi;                     // return the block number
            }
        }
        brelse(bp);
    }
    printf("balloc: out of blocks\n");
    return 0;                                    // 0 = failure
}</code></pre>

<p><strong>Step-by-step actions:</strong></p>
<ol>
  <li>Scan the bitmap block by block, looking for a bit that is 0 (free)</li>
  <li>When found: set that bit to 1 (mark allocated), call <code>log_write()</code> on the bitmap block, release it</li>
  <li>Zero the newly allocated block (<code>bzero()</code>) — prevents any previous file's data from leaking into the new block</li>
  <li>Return the block number</li>
</ol>

<p><strong>When the disk is full:</strong></p>
<p><code>balloc()</code> returns <strong>0</strong>. Block 0 is the boot block — it is never a valid data block address. Callers check for this:</p>
<pre><code class="language-c">addr = balloc(ip->dev);
if(addr == 0)
    return 0;   // propagate failure up to write() → returns 0 bytes written</code></pre>

</div>
</div>

<div class="var-question">
<div class="var-qnum">Q3 <span class="diff diff-easy">Easy</span></div>

What is <code>BPB</code> and how many bitmap blocks does xv6 need for a 200,000-block filesystem?

<div class="answer-content">

<p><strong>What BPB is:</strong></p>
<pre><code class="language-c">// kernel/fs.h
#define BPB           (BSIZE*8)   // Bits Per Block = 1024 * 8 = 8192</code></pre>

<p>One disk block (1024 bytes = 8192 bits) can track 8,192 disk blocks in the bitmap. <code>BPB</code> is used to calculate which bitmap block covers a given disk block:</p>
<pre><code class="language-c">#define BBLOCK(b, sb) ((b)/BPB + sb.bmapstart)
// BBLOCK(0, sb)    → first bitmap block
// BBLOCK(8192, sb) → second bitmap block
// BBLOCK(16384, sb)→ third bitmap block</code></pre>

<p><strong>Bitmap blocks needed for FSSIZE=200,000:</strong></p>
<pre><code class="language-c">bits_needed    = FSSIZE = 200,000
bytes_needed   = 200,000 / 8 = 25,000 bytes
blocks_needed  = ⌈25,000 / 1024⌉ = ⌈24.41...⌉ = 25 blocks</code></pre>

<p>This matches the <code>mkfs</code> output after the change:</p>
<pre><code class="language-c">nmeta 70 (boot, super, log blocks 30 inode blocks 13, bitmap blocks 25) blocks 199930 total 200000
//                                                                 ^^
//                                                        25 bitmap blocks
</code></pre>

<p><strong>For the original FSSIZE=2000:</strong></p>
<pre><code class="language-c">bits_needed  = 2000
bytes_needed = 2000 / 8 = 250 bytes
blocks_needed = ⌈250 / 1024⌉ = 1 block   // 250 bytes fits in one 1024-byte block</code></pre>

</div>
</div>

<div class="var-question">
<div class="var-qnum">Q4 <span class="diff diff-easy">Easy</span></div>

What does <code>bfree(dev, blockno)</code> do, and what is the only difference between a block that <code>bfree</code> just freed and one that has never been used?

<div class="answer-content">

<p><strong>What bfree() does:</strong></p>
<pre><code class="language-c">// kernel/fs.c — bfree() behaviour:
static void bfree(int dev, uint b)
{
    struct buf *bp;
    int bi, m;

    bp = bread(dev, BBLOCK(b, sb));   // load the bitmap block covering block b
    bi = b % BPB;
    m = 1 &lt;&lt; (bi % 8);
    if((bp->data[bi/8] &amp; m) == 0)
        panic("freeing free block");  // double-free detected
    bp->data[bi/8] &amp;= ~m;            // clear the bit: 1 → 0 (mark free)
    log_write(bp);
    brelse(bp);
    // NOTE: does NOT zero the block's contents
}</code></pre>

<p><code>bfree()</code> only clears the bitmap bit — it does not touch the block's data on disk. The block's old data remains exactly as it was.</p>

<p><strong>The only difference between a bfree'd block and a never-used block:</strong></p>

<table>
  <tr><th>Property</th><th>bfree'd block</th><th>Never-used block</th></tr>
  <tr><td>Bitmap bit</td><td>0 (free) — identical</td><td>0 (free) — identical</td></tr>
  <tr><td>Block contents on disk</td><td>Contains old data from the previous file</td><td>Contains zeros (mkfs zeroes the disk image)</td></tr>
</table>

<p><strong>Why this matters:</strong></p>
<p><code>balloc()</code> always calls <code>bzero()</code> on the block it returns before giving it to the caller. This ensures that regardless of whether the block was previously used, the caller always receives a zeroed block. Without <code>bzero()</code>, a newly allocated block could contain a previous file's pointer values — which in the context of a map block would cause <code>bmap()</code> to follow stale pointers to data that does not belong to the current file.</p>

</div>
</div>

---

## Medium

<div class="var-question">
<div class="var-qnum">Q5 <span class="diff diff-med">Medium</span></div>

How many buffer cache slots does a single call to <code>bmap()</code> hold simultaneously at its peak, and why does this matter for a system with NBUF=30?

<div class="answer-content">

<p><strong>Peak simultaneous buffer holdings in bmap():</strong></p>
<pre><code class="language-c">// Direct block path: peak = 0 held (just reads ip->addrs[])
// No bread() calls needed.

// Singly-indirect path: peak = 1 held simultaneously
bp = bread(dev, ip->addrs[NDIRECT]);   // hold 1
// ... read a[bn] ...
brelse(bp);                             // release: back to 0

// Doubly-indirect path: peak = 1 held simultaneously (by design)
bp = bread(dev, ip->addrs[NDIRECT+1]); // hold 1 (master map)
// ... read a[idx1] ...
brelse(bp);                             // release: back to 0  ← CRITICAL

bp = bread(dev, addr);                 // hold 1 (secondary map)
// ... read a[idx2] ...
brelse(bp);                             // release: back to 0</code></pre>

<p><strong>Why the maximum is 1, not 2:</strong></p>
<p>The bmap() implementation is carefully written to release the master map buffer (<code>brelse(bp)</code>) before calling <code>bread()</code> for the secondary map. This ensures the peak never exceeds 1 buffer held at any time within a single bmap() call. This is deliberate — it prevents bmap() from consuming two slots from the 30-slot cache simultaneously.</p>

<p><strong>Why this matters with NBUF=30:</strong></p>
<p>In a running xv6 system, many paths hold buffers simultaneously:</p>

<table>
  <tr><th>Caller</th><th>Buffers held</th></tr>
  <tr><td>Log layer (<code>end_op</code>)</td><td>Up to LOGSIZE=30 in theory, but commits are serialised</td></tr>
  <tr><td><code>itrunc()</code> doubly-indirect cleanup</td><td>2 simultaneously: <code>bp1</code> (master) + <code>bp2</code> (secondary)</td></tr>
  <tr><td>Directory operations</td><td>1–2 buffers for directory blocks</td></tr>
  <tr><td>balloc()</td><td>1 buffer (bitmap block)</td></tr>
</table>

<p>If bmap() held 2 buffers simultaneously, a single write operation that calls bmap() + balloc() + iupdate() could hold 4 buffers. With 8 concurrent processes, that's 32 — exceeding NBUF=30 and causing a panic. By keeping bmap() at peak=1, the design stays safely within the buffer budget.</p>

</div>
</div>

<div class="var-question">
<div class="var-qnum">Q6 <span class="diff diff-med">Medium</span></div>

In <code>itrunc()</code>, the doubly-indirect cleanup holds <code>bp1</code> (master map) while reading and releasing multiple <code>bp2</code> (secondary map) buffers inside the loop. Does this violate the "one buffer at a time" rule from bmap()? Explain why it is safe here.

<div class="answer-content">

<p><strong>The itrunc() doubly-indirect cleanup:</strong></p>
<pre><code class="language-c">bp1 = bread(ip->dev, ip->addrs[NDIRECT+1]);   // hold bp1 (master map)
a1  = (uint*)bp1->data;

for(int i = 0; i &lt; NINDIRECT; i++){
    if(a1[i]){
        bp2 = bread(ip->dev, a1[i]);           // hold bp2 (secondary map) — now 2 held!
        a2  = (uint*)bp2->data;

        for(int j = 0; j &lt; NINDIRECT; j++){
            if(a2[j]) bfree(ip->dev, a2[j]);
        }

        brelse(bp2);                           // release bp2 — back to 1 held
        bfree(ip->dev, a1[i]);
    }
}

brelse(bp1);                                   // release bp1 — back to 0 held</code></pre>

<p><strong>Is this a violation?</strong></p>
<p>Yes — <code>itrunc()</code> holds 2 buffers simultaneously at the peak of each inner loop iteration. This is intentional and safe for two reasons:</p>

<p><strong>Reason 1 — itrunc() is called rarely and in a controlled context:</strong></p>
<p>itrunc() is only called when a file is being fully deleted or explicitly truncated. It is never called from a hot path where many concurrent processes might be calling it simultaneously. The brief 2-buffer window is bounded and well-understood.</p>

<p><strong>Reason 2 — The two buffers are guaranteed to be different blocks:</strong></p>
<p><code>bp1</code> is the master map block. <code>bp2</code> is a secondary map block. These are never the same physical block — the master map holds pointers to secondary maps, not to itself. The buffer cache allows two different blocks to be held simultaneously; it only panics when all 30 slots are locked by threads that are all waiting for a 31st slot.</p>

<p><strong>What would actually be dangerous:</strong></p>
<p>Calling <code>bread()</code> for a block that is already locked by the same or another thread that is blocked waiting for it — that creates a deadlock. <code>itrunc()</code> avoids this because it always releases <code>bp2</code> before looping back to acquire the next <code>bp2</code>.</p>

</div>
</div>

<div class="var-question">
<div class="var-qnum">Q7 <span class="diff diff-med">Medium</span></div>

Why does <code>balloc()</code> call <code>bzero()</code> on the newly allocated block before returning it? What specific bug in <code>bmap()</code> would occur if it did not?

<div class="answer-content">

<p><strong>What bzero() does:</strong></p>
<pre><code class="language-c">static void bzero(int dev, int bno)
{
    struct buf *bp = bread(dev, bno);
    memset(bp->data, 0, BSIZE);   // zero all 1024 bytes
    log_write(bp);
    brelse(bp);
}</code></pre>

<p>Every block returned by <code>balloc()</code> is guaranteed to contain all zeros.</p>

<p><strong>The bug that occurs without bzero() — in the doubly-indirect bmap() path:</strong></p>
<pre><code class="language-c">// Suppose block 500 was previously used as a secondary map for a different file.
// That file was deleted: bfree() cleared the bitmap bit but left block 500's data intact.
// Block 500 still contains: [301, 417, 82, 0, 0, ...]  (old pointers)

// Now bmap() allocates block 500 as the NEW master map for our file:
addr = balloc(ip->dev);   // returns 500 — NOT zeroed
ip->addrs[NDIRECT+1] = 500;

bp = bread(ip->dev, 500);
a  = (uint*)bp->data;
// a[] = [301, 417, 82, 0, 0, ...]  ← stale data from old file!

// bmap() asks: "do I need a secondary map for idx1=0?"
if((addr = a[idx1]) == 0)  // a[0] = 301 — NOT zero!
    addr = balloc(...);    // ← SKIPPED! No new secondary map allocated.

// bmap() loads block 301 as the secondary map for our file.
// Block 301 belongs to the old deleted file and contains OLD data block pointers.
// Any write through our new file goes to the old file's data blocks — 
// silently overwriting whatever is stored in block 301's pointers.</code></pre>

<p><strong>Why zeroing is the correct fix:</strong></p>
<p>After <code>balloc()</code> zeros the new master map block, every entry is 0. The <code>if((addr = a[idx1]) == 0)</code> check correctly identifies "no secondary map allocated yet" and allocates a fresh one. There is no contamination from the block's previous life.</p>

</div>
</div>

<div class="var-question">
<div class="var-qnum">Q8 <span class="diff diff-med">Medium</span></div>

What is the <code>IBLOCK</code> macro and how does xv6 find the disk block that contains a specific inode number?

<div class="answer-content">

<p><strong>The IBLOCK macro:</strong></p>
<pre><code class="language-c">// kernel/fs.h
#define IPB          (BSIZE / sizeof(struct dinode))  // inodes per block
// sizeof(struct dinode) = 64 bytes → IPB = 1024/64 = 16 inodes per block

#define IBLOCK(i, sb) ((i) / IPB + sb.inodestart)
// i = inode number
// sb.inodestart = block number where the inode region begins (block 32 in xv6)</code></pre>

<p><strong>Finding inode number 47:</strong></p>
<pre><code class="language-c">// Which inode block?
block = 47 / 16 + 32 = 2 + 32 = block 34

// Which slot within that block?
offset = 47 % 16 = 15   (zero-based: the 16th inode in block 34)

// In code:
struct buf *bp = bread(dev, IBLOCK(47, sb));
struct dinode *dip = (struct dinode*)bp->data + (47 % IPB);
// dip now points to the 16th dinode in block 34</code></pre>

<p><strong>The inode region layout:</strong></p>

<table>
  <tr><th>Inode block (disk block)</th><th>Inode numbers it holds</th></tr>
  <tr><td>32 (inodestart)</td><td>0–15</td></tr>
  <tr><td>33</td><td>16–31</td></tr>
  <tr><td>34</td><td>32–47</td></tr>
  <tr><td>…</td><td>…</td></tr>
  <tr><td>44 (last inode block)</td><td>192–207</td></tr>
</table>

<p>With 13 inode blocks and 16 inodes per block, xv6 supports up to 13 × 16 = 208 inodes total. <code>NINODES = 200</code> in <code>param.h</code> — 8 slots are unused. Inode 0 is reserved (never allocated); inode 1 is the root directory.</p>

</div>
</div>

---

## Hard

<div class="var-question">
<div class="var-qnum">Q9 <span class="diff diff-hard">Hard</span></div>

Trace the complete sequence of buffer cache operations that occur when <code>itrunc()</code> frees a file that used exactly 300 doubly-indirect data blocks (the first secondary map is full, the second has 44 entries). Count every <code>bread()</code>, <code>brelse()</code>, and <code>bfree()</code> call.

<div class="answer-content">

<p><strong>Setup:</strong> File uses logical blocks 267–566 (300 doubly-indirect blocks). Secondary map #0 (idx1=0) is full with 256 entries. Secondary map #1 (idx1=1) has 44 entries (blocks 0–43).</p>

<p><strong>Outer loop — load master map:</strong></p>
<pre><code class="language-c">bp1 = bread(dev, master_map_block);   // (1) bread: load master map
a1  = (uint*)bp1->data;
// a1[0] = addr of secondary map #0 (non-zero)
// a1[1] = addr of secondary map #1 (non-zero)
// a1[2..255] = 0 (never allocated)</code></pre>

<p><strong>Inner loop iteration i=0 (secondary map #0, 256 data blocks):</strong></p>
<pre><code class="language-c">bp2 = bread(dev, a1[0]);              // (2) bread: load secondary map #0
a2  = (uint*)bp2->data;
// a2[0..255] all non-zero

for j=0..255:
    bfree(dev, a2[j]);                // (3)–(258) bfree: 256 × bfree, each calls bread+brelse on bitmap

brelse(bp2);                          // (259) brelse: release secondary map #0
bfree(dev, a1[0]);                    // (260) bfree: free the secondary map #0 block itself</code></pre>

<p><strong>Inner loop iteration i=1 (secondary map #1, 44 data blocks):</strong></p>
<pre><code class="language-c">bp2 = bread(dev, a1[1]);              // (261) bread: load secondary map #1
a2  = (uint*)bp2->data;
// a2[0..43] non-zero, a2[44..255] = 0

for j=0..43:
    bfree(dev, a2[j]);                // (262)–(305) bfree: 44 × bfree

brelse(bp2);                          // (306) brelse: release secondary map #1
bfree(dev, a1[1]);                    // (307) bfree: free the secondary map #1 block itself</code></pre>

<p><strong>Iterations i=2..255: a1[i]=0, loop body skipped.</strong></p>

<p><strong>After the outer loop:</strong></p>
<pre><code class="language-c">brelse(bp1);                          // (308) brelse: release master map
bfree(dev, ip->addrs[NDIRECT+1]);     // (309) bfree: free master map block itself
ip->addrs[NDIRECT+1] = 0;</code></pre>

<p><strong>Summary:</strong></p>

<table>
  <tr><th>Operation</th><th>Count</th><th>What</th></tr>
  <tr><td><code>bread()</code></td><td>3</td><td>1 master map + 2 secondary maps</td></tr>
  <tr><td><code>brelse()</code></td><td>3</td><td>1 master map + 2 secondary maps</td></tr>
  <tr><td><code>bfree()</code> on data blocks</td><td>300</td><td>256 + 44 data blocks</td></tr>
  <tr><td><code>bfree()</code> on map blocks</td><td>3</td><td>2 secondary maps + 1 master map</td></tr>
  <tr><td><strong>Total bfree()</strong></td><td><strong>303</strong></td><td>300 data + 2 secondary + 1 master</td></tr>
</table>

<p>Each <code>bfree()</code> call internally calls <code>bread()</code> + <code>brelse()</code> on the bitmap block — those are additional buffer cache operations not counted above. The bitmap block for our data region is likely the same block for many of the 300 data blocks (since they're in adjacent block numbers), so the buffer cache's LRU will often serve the bitmap block from cache rather than re-reading it from disk.</p>

</div>
</div>

<div class="var-question">
<div class="var-qnum">Q10 <span class="diff diff-hard">Hard</span></div>

A student adds a print statement inside <code>bmap()</code> that calls <code>printf("[bmap] bn=%d → block=%d\n", bn_original, result)</code>. What unexpected side effect can this cause in xv6, and why?

<div class="answer-content">

<p><strong>What printf() does in xv6 kernel space:</strong></p>
<p>In xv6, <code>printf()</code> in the kernel ultimately calls <code>consputc()</code> which acquires the console lock (<code>cons.lock</code>) and writes characters to the UART output device. This is a <strong>blocking, lock-acquiring operation</strong>.</p>

<p><strong>The problem — bmap() is called while holding inode locks:</strong></p>
<pre><code class="language-c">// Typical call chain leading to bmap():
ilock(ip);           // ← acquires ip->lock (a sleeplock)
    writei(ip, ...);
        bmap(ip, bn);
            printf("[bmap] ...");   // ← tries to acquire cons.lock</code></pre>

<p>If <code>cons.lock</code> is held by another CPU (e.g. the shell is printing output), the kernel's <code>printf()</code> call spins on or sleeps waiting for the console lock — while holding <code>ip->lock</code>.</p>

<p><strong>Specific bugs this can trigger:</strong></p>

<table>
  <tr><th>Scenario</th><th>Bug</th></tr>
  <tr><td>Another process calls <code>open()</code> on the same file while ilock is held by the printf-waiting thread</td><td>That process blocks on <code>ilock()</code> — not a bug, expected behaviour. But if that process holds cons.lock (unlikely), circular wait → deadlock.</td></tr>
  <tr><td>bmap() is called from interrupt context (e.g. disk interrupt handler)</td><td>printf() → consputc() → acquiresleep() → sleep() — you cannot sleep in interrupt context. <code>panic("sleep")</code> or undefined behaviour.</td></tr>
  <tr><td>High-frequency bmap() calls (bigfile writes 6580 blocks)</td><td>6580 × printf() calls flood the UART. Each call holds the console lock for milliseconds. Overall runtime increases by seconds — the grader may time out.</td></tr>
  <tr><td>bmap() called during boot with partial filesystem state</td><td>printf() may try to use a console that has not yet been fully initialised.</td></tr>
</table>

<p><strong>The correct way to add debugging to bmap():</strong></p>
<pre><code class="language-c">// Use a conditional compile flag, and only print OUTSIDE of lock-holding paths:
#ifdef BMAP_DEBUG
    // Store values to a circular ring buffer in RAM, print after releasing locks
#endif

// Or: use the existing cprintf() with care, only when no sleeplocks are held</code></pre>

</div>
</div>

<div class="var-question">
<div class="var-qnum">Q11 <span class="diff diff-hard">Hard</span></div>

Construct a scenario where a correct implementation of <code>bmap()</code> and <code>itrunc()</code> still causes <code>panic("balloc: out of blocks")</code> without any bug in the student's code. Explain exactly what invariant is violated and by which part of xv6.

<div class="answer-content">

<p><strong>The scenario — log transaction too large:</strong></p>
<pre><code class="language-c">// A single write() to bigfile may call bmap() for each of the 6580 blocks.
// Each bmap() call that allocates a new doubly-indirect block calls:
//   balloc() for master map    → log_write(bitmap_block)
//   log_write(master_map)
//   balloc() for secondary map → log_write(bitmap_block)
//   log_write(secondary_map)
//   balloc() for data block    → log_write(bitmap_block)
//   log_write(data_block... wait, data blocks are NOT log_write'd by bmap)

// For the first 6580 blocks, bmap() also calls log_write() on:
//   - 1 master map block
//   - 26 secondary map blocks (one per 256 data blocks)
//   - ~27 bitmap block updates (each balloc log_writes the bitmap block)

// Total log_write() calls ≈ 1 + 26 + 27 = 54 dirty buffers in one end_op() call</code></pre>

<p><strong>The invariant violated — MAXOPBLOCKS:</strong></p>
<pre><code class="language-c">// kernel/param.h
#define MAXOPBLOCKS  10   // max # of blocks any FS op writes
#define LOGSIZE      (MAXOPBLOCKS*3+3)  // = 33 log slots</code></pre>

<p><code>begin_op()</code> checks that the pending transaction will not exceed <code>LOGSIZE</code> blocks. If <code>writei()</code> for a large write is wrapped in a single <code>begin_op()</code>/<code>end_op()</code>, the log can overflow:</p>
<pre><code class="language-c">begin_op();                      // starts transaction
writei(ip, buf, 0, 6580*1024);  // ← triggers 54+ log_write() calls
end_op();                        // tries to commit — log has 54 dirty entries
                                  // but LOGSIZE = 33 → panic("log write too big")</code></pre>

<p><strong>How xv6 actually avoids this (why bigfile works):</strong></p>
<p><code>writei()</code> in xv6 does NOT hold a single transaction across all 6580 block writes. Each call to <code>bwrite()</code> within <code>writei()</code> is part of the user's transaction started by the <code>write()</code> syscall, which in turn calls <code>begin_op()</code> once per syscall. The <code>write()</code> syscall for <code>bigfile</code> calls <code>writei()</code> for one <code>BSIZE</code> buffer at a time per iteration of the for loop — so each loop iteration is one <code>begin_op()</code>/<code>end_op()</code> pair covering at most a few <code>log_write()</code> calls, well within <code>LOGSIZE</code>.</p>

<p><strong>The panic actually triggered — when a student wraps too many operations in one transaction:</strong></p>
<pre><code class="language-c">// Student's buggy test:
begin_op();
for(int i = 0; i &lt; 6580; i++){
    bmap(ip, i);      // each bmap() may call log_write() 3 times
    // → 6580 × 3 = 19,740 log_write() calls in one transaction
}
end_op();
// begin_op() would have caught this: panic("begin_op: log too big")</code></pre>

<p>The result looks like a disk-full error but is actually a log overflow — distinguishable because the panic message says "log" rather than "balloc".</p>

</div>
</div>
