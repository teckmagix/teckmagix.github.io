---
layout: page
title: "Lab Predictions - lab3-f7n2"
lab: lab3
description: "Exam predictions for Lab3-w8: bmap() navigation, itrunc() cleanup, TEST_BLOCKS, and define ordering."
---

## Easy

<div class="var-question">
<div class="var-qnum">Q1 <span class="diff diff-easy">Easy</span></div>

The grader expects <code>wrote 6580 blocks</code>, not <code>wrote 65803 blocks</code>. Why, and what line in <code>bigfile.c</code> controls this?

<div class="answer-content">

<p><strong>The controlling define:</strong></p>
<pre><code class="language-c">#define TEST_BLOCKS (65803/10)
// evaluates to 6580 (integer division)</code></pre>

<p><strong>Why the grader uses one-tenth:</strong></p>
<p>Writing 65,803 blocks of 1 KB each requires the kernel to allocate and navigate the full doubly-indirect tree. On Gradescope's shared infrastructure this takes 2–30 minutes depending on the host machine. At one-tenth — 6,580 blocks — the test still exercises the doubly-indirect path (the singly-indirect range covers only blocks 11–266, so block 267 onward requires doubly-indirect), but completes quickly enough for automated grading.</p>

<p><strong>The loop that enforces it:</strong></p>
<pre><code class="language-c">for (i = 0; i &lt; TEST_BLOCKS; i++){
    *(int*)buf = blocks;          // store current block count in buffer
    int cc = write(fd, buf, sizeof(buf));
    if(cc &lt;= 0)
        break;
    blocks++;
    if (blocks % 100 == 0)
        printf(".");              // print a dot every 100 blocks
}

printf("\nwrote %d blocks\n", blocks);
if(blocks != TEST_BLOCKS) {
    printf("bigfile: file is too small\n");
    exit(-1);
}</code></pre>

<p><strong>Why the original infinite <code>while(1)</code> loop fails the grader:</strong></p>
<p>The original loop writes until <code>write()</code> returns ≤ 0 (disk full or limit hit). With the new 65,803-block limit it writes all 65,803 blocks and prints <code>wrote 65803 blocks</code>. The grader's pattern match expects the string <code>wrote 6580 blocks</code> — a mismatch causes the test to fail even though the implementation is correct.</p>

</div>
</div>

<div class="var-question">
<div class="var-qnum">Q2 <span class="diff diff-easy">Easy</span></div>

Why does the four-line define block in <code>kernel/fs.h</code> need to appear in exactly this order? What breaks if you put <code>MAXFILE</code> first?

<div class="answer-content">

<p><strong>The correct order:</strong></p>
<pre><code class="language-c">#define NDIRECT    11
#define NINDIRECT  (BSIZE / sizeof(uint))          // depends on nothing above
#define NDINDIRECT (NINDIRECT * NINDIRECT)          // depends on NINDIRECT
#define MAXFILE    (NDIRECT + NINDIRECT + NDINDIRECT) // depends on all three</code></pre>

<p><strong>C preprocessor substitution — why order matters:</strong></p>
<p>The C preprocessor expands macros textually at the point of use. If <code>MAXFILE</code> is defined before <code>NDINDIRECT</code>, the preprocessor sees:</p>
<pre><code class="language-c">#define MAXFILE (NDIRECT + NINDIRECT + NDINDIRECT)
// At this point NDINDIRECT is not yet defined.
// The compiler sees: (11 + 256 + NDINDIRECT) — NDINDIRECT is an undefined identifier.
// Result: compilation error — "NDINDIRECT undeclared"</code></pre>

<p><strong>Dependency chain:</strong></p>

<table>
  <tr><th>Define</th><th>Depends on</th><th>Value</th></tr>
  <tr><td><code>NDIRECT 11</code></td><td>Nothing</td><td>11</td></tr>
  <tr><td><code>NINDIRECT</code></td><td><code>BSIZE</code> (already defined earlier as 1024), <code>sizeof(uint)</code></td><td>256</td></tr>
  <tr><td><code>NDINDIRECT</code></td><td><code>NINDIRECT</code></td><td>65,536</td></tr>
  <tr><td><code>MAXFILE</code></td><td><code>NDIRECT</code>, <code>NINDIRECT</code>, <code>NDINDIRECT</code></td><td>65,803</td></tr>
</table>

<p><strong>Why changing <code>NDIRECT</code> from 12 to 11 does not change the array size:</strong></p>
<pre><code class="language-c">// Before: NDIRECT=12 → addrs[NDIRECT+1] = addrs[13]
// After:  NDIRECT=11 → addrs[NDIRECT+2] = addrs[13]
// Both expressions evaluate to 13 — array size unchanged.
// On-disk inode layout is bit-for-bit identical — no filesystem corruption.</code></pre>

</div>
</div>

<div class="var-question">
<div class="var-qnum">Q3 <span class="diff diff-easy">Easy</span></div>

What is <code>FSSIZE</code> and why must it be changed to 200,000 for this lab?

<div class="answer-content">

<p><strong>What FSSIZE controls:</strong></p>
<pre><code class="language-c">// kernel/param.h
#define FSSIZE       200000   // was 2000</code></pre>

<p><code>FSSIZE</code> is passed to <code>mkfs</code> — the tool that builds the xv6 disk image at compile time. It specifies the total number of 1 KB blocks in the entire disk image. This includes metadata blocks (boot, superblock, log, inodes, bitmap) as well as data blocks.</p>

<p><strong>Why 2000 is too small:</strong></p>

<table>
  <tr><th>Scenario</th><th>Blocks needed</th></tr>
  <tr><td>bigfile data blocks (65,803)</td><td>65,803</td></tr>
  <tr><td>Master map block (1 per large file)</td><td>1</td></tr>
  <tr><td>Secondary map blocks (up to 256)</td><td>256</td></tr>
  <tr><td>Total for one max-size file</td><td>66,060</td></tr>
  <tr><td>Original FSSIZE</td><td>2,000</td></tr>
</table>

<p>A 65,803-block file simply does not fit in a 2,000-block disk. <code>balloc()</code> would exhaust all available blocks and return 0 long before the file is complete — <code>bmap()</code> would return 0, <code>write()</code> would return 0, and <code>bigfile</code> would exit with "file is too small".</p>

<p><strong>What you see after the change:</strong></p>
<pre><code class="language-c">// mkfs output line after make:
nmeta 70 (boot, super, log blocks 30 inode blocks 13, bitmap blocks 25) blocks 199930 total 200000
//                                                                                      ^^^^^^
// 199,930 data blocks available — more than enough for a 65,803-block file.</code></pre>

</div>
</div>

<div class="var-question">
<div class="var-qnum">Q4 <span class="diff diff-easy">Easy</span></div>

What does <code>log_write(bp)</code> do and why must it be called every time a map block is modified?

<div class="answer-content">

<p><strong>What log_write does:</strong></p>
<p><code>log_write(bp)</code> marks a buffer as "dirty" and records it in the kernel's write-ahead log. It does <strong>not</strong> write the buffer to its final disk location immediately. Instead, when <code>end_op()</code> is called, the log flushes all dirty buffers to the log section on disk first, writes a commit record, and only then copies them to their real locations. If the machine crashes after the commit record is written, the log can replay all writes on reboot. If it crashes before, the partial writes are discarded.</p>

<p><strong>Where it appears in bmap():</strong></p>
<pre><code class="language-c">// Level 2: allocate secondary map, update master map entry
if((addr = a[idx1]) == 0){
    addr = balloc(ip->dev);
    if(addr){
        a[idx1] = addr;          // ← modify the master map buffer in RAM
        log_write(bp);           // ← tell the log this buffer is dirty
    }
}
brelse(bp);   // ← release the master map buffer

// Level 3: allocate data block, update secondary map entry
if((addr = a[idx2]) == 0){
    addr = balloc(ip->dev);
    if(addr){
        a[idx2] = addr;          // ← modify the secondary map buffer in RAM
        log_write(bp);           // ← tell the log this buffer is dirty
    }
}
brelse(bp);</code></pre>

<p><strong>What happens if you omit log_write:</strong></p>

<table>
  <tr><th>Scenario</th><th>Consequence</th></tr>
  <tr><td>Machine stays up</td><td>Works — the buffer cache eventually writes the block back anyway</td></tr>
  <tr><td>Machine crashes mid-operation</td><td>The map block update is lost — the inode points to a secondary map block that has a zero entry, corrupting the file</td></tr>
  <tr><td>Running the grader</td><td>Likely passes — but the implementation is incorrect by design</td></tr>
</table>

<p><strong>Key rule:</strong> Every time you write a new pointer into a map buffer (master or secondary), call <code>log_write(bp)</code> before <code>brelse(bp)</code>. Read-only access to a buffer does not require <code>log_write()</code>.</p>

</div>
</div>

---

## Medium

<div class="var-question">
<div class="var-qnum">Q5 <span class="diff diff-med">Medium</span></div>

Trace <code>bmap()</code> for logical block numbers <code>bn = 0</code>, <code>bn = 11</code>, <code>bn = 267</code>, and <code>bn = 268</code>. State which range each falls in and show the subtraction steps.

<div class="answer-content">

<p><strong>Constants:</strong> <code>NDIRECT=11</code>, <code>NINDIRECT=256</code>, <code>NDINDIRECT=65536</code></p>

<p><strong>bn = 0 — direct block:</strong></p>
<pre><code class="language-c">if(bn &lt; NDIRECT)           // 0 &lt; 11? Yes
    return ip->addrs[0];   // first direct block — 1 disk read total</code></pre>

<p><strong>bn = 11 — first singly-indirect block:</strong></p>
<pre><code class="language-c">// bn=11 is NOT less than NDIRECT=11, so we fall through
bn -= NDIRECT;             // 11 - 11 = 0
if(bn &lt; NINDIRECT)         // 0 &lt; 256? Yes
// Load indirect block from ip->addrs[NDIRECT]=ip->addrs[11]
// Return indirect_block[0]  ← 2 disk reads total</code></pre>

<p><strong>bn = 267 — last singly-indirect block:</strong></p>
<pre><code class="language-c">bn -= NDIRECT;             // 267 - 11 = 256... wait
// 267 - 11 = 256. Is 256 &lt; NINDIRECT(256)? No — falls to doubly-indirect.
// Correction: last singly-indirect is bn=266.
// 266 - 11 = 255. Is 255 &lt; 256? Yes → singly-indirect.
// bn=267: 267-11=256. Not &lt;256 → falls through.
bn -= NINDIRECT;           // 256 - 256 = 0
if(bn &lt; NDINDIRECT)        // 0 &lt; 65536? Yes
// First doubly-indirect block. idx1=0/256=0, idx2=0%256=0</code></pre>

<p><strong>bn = 268 — second doubly-indirect block:</strong></p>
<pre><code class="language-c">bn -= NDIRECT;             // 268 - 11 = 257
// 257 &lt; 256? No
bn -= NINDIRECT;           // 257 - 256 = 1
// 1 &lt; 65536? Yes → doubly-indirect
uint idx1 = 1 / 256;       // = 0  → still secondary map #0
uint idx2 = 1 % 256;       // = 1  → slot 1 of secondary map #0
// Path: ip->addrs[12] → master map[0] → secondary map[1] → data block</code></pre>

<p><strong>Summary table:</strong></p>

<table>
  <tr><th>bn</th><th>After subtracting NDIRECT</th><th>After subtracting NINDIRECT</th><th>Range</th><th>Disk reads</th></tr>
  <tr><td>0</td><td>—</td><td>—</td><td>Direct</td><td>1</td></tr>
  <tr><td>11</td><td>0</td><td>—</td><td>Singly-indirect</td><td>2</td></tr>
  <tr><td>266</td><td>255</td><td>—</td><td>Singly-indirect (last)</td><td>2</td></tr>
  <tr><td>267</td><td>256</td><td>0</td><td>Doubly-indirect (first)</td><td>3</td></tr>
  <tr><td>268</td><td>257</td><td>1</td><td>Doubly-indirect</td><td>3</td></tr>
</table>

</div>
</div>

<div class="var-question">
<div class="var-qnum">Q6 <span class="diff diff-med">Medium</span></div>

In <code>bmap()</code>, why is <code>brelse(bp)</code> called on the master map buffer <em>before</em> calling <code>bread()</code> for the secondary map buffer? What panic results if you hold both simultaneously?

<div class="answer-content">

<p><strong>The critical ordering in bmap():</strong></p>
<pre><code class="language-c">// Load master map (level 1)
bp = bread(ip->dev, addr);          // bp is now LOCKED
a = (uint*)bp->data;

uint idx1 = bn / NINDIRECT;
uint idx2 = bn % NINDIRECT;

if((addr = a[idx1]) == 0){
    addr = balloc(ip->dev);
    if(addr){ a[idx1] = addr; log_write(bp); }
}
brelse(bp);                         // ← MUST release before next bread()

if(addr == 0) return 0;

// Load secondary map (level 2)
bp = bread(ip->dev, addr);          // bp re-used for secondary map
a = (uint*)bp->data;</code></pre>

<p><strong>Why simultaneous holding causes a panic:</strong></p>
<p>The buffer cache in xv6 has only <strong>NBUF = 30</strong> slots (defined in <code>kernel/param.h</code>). Each call to <code>bread()</code> locks one slot. If all 30 slots are locked simultaneously — whether by one thread or multiple threads — the next <code>bread()</code> call in <code>bget()</code> cannot find a free slot and calls:</p>
<pre><code class="language-c">panic("bget: no buffers");</code></pre>

<p><strong>How this happens in practice:</strong></p>
<p>The kernel calls <code>bmap()</code> during every file read and write. If <code>bmap()</code> holds the master map buffer while calling <code>bread()</code> for the secondary map, that is 2 locked slots per call. With concurrent processes each holding 2 slots, the 30-slot cache exhausts after only 15 simultaneous calls — easily reached in a system with multiple processes and the logging layer also using buffers.</p>

<p><strong>The correct pattern:</strong></p>
<pre><code class="language-c">// Extract the address you need THEN immediately release the buffer.
uint addr_of_secondary = a[idx1];   // read the address
brelse(bp);                          // release master map buffer NOW
// Now safe to call bread() for the secondary map.</code></pre>

</div>
</div>

<div class="var-question">
<div class="var-qnum">Q7 <span class="diff diff-med">Medium</span></div>

In <code>itrunc()</code>, why must <code>brelse(bp2)</code> be called before <code>bfree(ip-&gt;dev, a1[i])</code>, and why must <code>brelse(bp1)</code> be called before <code>bfree(ip-&gt;dev, ip-&gt;addrs[NDIRECT+1])</code>?

<div class="answer-content">

<p><strong>The cleanup structure:</strong></p>
<pre><code class="language-c">bp1 = bread(ip->dev, ip->addrs[NDIRECT+1]);   // load master map
a1 = (uint*)bp1->data;

for(int i = 0; i &lt; NINDIRECT; i++){
    if(a1[i]){
        bp2 = bread(ip->dev, a1[i]);           // load secondary map
        a2 = (uint*)bp2->data;

        for(int j = 0; j &lt; NINDIRECT; j++){
            if(a2[j])
                bfree(ip->dev, a2[j]);         // free each data block
        }

        brelse(bp2);                           // ← release BEFORE freeing
        bfree(ip->dev, a1[i]);                 // free secondary map block
    }
}

brelse(bp1);                                   // ← release BEFORE freeing
bfree(ip->dev, ip->addrs[NDIRECT+1]);          // free master map block
ip->addrs[NDIRECT+1] = 0;</code></pre>

<p><strong>Why brelse must precede bfree:</strong></p>
<p>The buffer cache stores <strong>block number → buffer</strong> mappings. When a buffer for block N is locked (held via <code>bread()</code>), the cache entry for block N remains valid. If you call <code>bfree(dev, N)</code> while the buffer is still locked:</p>
<ol>
  <li><code>bfree()</code> clears bit N in the bitmap — marking block N as free</li>
  <li>Another process calls <code>balloc()</code>, finds bit N is 0 (free), and allocates block N for a completely different file</li>
  <li>That process starts writing new data to block N's buffer</li>
  <li>You still hold the old buffer for block N — you are now looking at another file's data</li>
  <li>Any subsequent <code>brelse()</code> or cache eviction writes your stale buffer back, corrupting the new file</li>
</ol>

<p>Releasing the buffer first ensures the cache entry is invalidated before the block is marked free and potentially reallocated.</p>

</div>
</div>

<div class="var-question">
<div class="var-qnum">Q8 <span class="diff diff-med">Medium</span></div>

What does <code>ip-&gt;addrs[NDIRECT+1] = 0</code> do at the end of the doubly-indirect cleanup in <code>itrunc()</code>? What happens at the next file write if this line is missing?

<div class="answer-content">

<p><strong>What the line does:</strong></p>
<pre><code class="language-c">bfree(ip->dev, ip->addrs[NDIRECT+1]);   // block returned to free pool
ip->addrs[NDIRECT+1] = 0;               // pointer cleared in the inode</code></pre>

<p>This clears the doubly-indirect pointer slot in the inode's <code>addrs[]</code> array. After freeing the master map block, the inode must no longer hold its address — the block now belongs to the free pool and could be allocated to another file at any moment.</p>

<p><strong>What happens if the line is missing — scenario:</strong></p>
<pre><code class="language-c">// File A is deleted. itrunc() frees all blocks but does NOT zero addrs[NDIRECT+1].
// Suppose the freed master map block was at disk block 500.
// ip->addrs[NDIRECT+1] still contains 500.

// The inode is written back to disk (iupdate) with addrs[NDIRECT+1] = 500.

// A new file B is created. balloc() finds block 500 is free (bitmap bit = 0).
// balloc() allocates block 500 to file B and writes file B's data there.

// File A's inode is still on disk with addrs[NDIRECT+1] = 500.
// If a process still holds a reference to File A's inode:
//   bmap() on File A follows addrs[NDIRECT+1] → block 500 → reads File B's data!
//   Any write through File A overwrites File B's data — silent corruption.</code></pre>

<p><strong>Why the inode persists after itrunc():</strong></p>
<p><code>itrunc()</code> frees the data blocks but does not free the inode itself — the inode stays allocated as long as <code>ip->nlink > 0</code> or a process has it open. Zeroing all <code>addrs[]</code> entries ensures that subsequent access through the inode hits the "block 0 → not allocated" path in <code>bmap()</code> rather than following a stale pointer.</p>

</div>
</div>

---

## Hard

<div class="var-question">
<div class="var-qnum">Q9 <span class="diff diff-hard">Hard</span></div>

Explain the full three-level allocation path in <code>bmap()</code> for a brand-new file where all three levels are absent (all pointers are zero). Trace every <code>balloc</code>, <code>bread</code>, <code>log_write</code>, and <code>brelse</code> call for logical block number <code>bn = 267</code> (the first doubly-indirect block).

<div class="answer-content">

<p><strong>Starting state:</strong> <code>ip->addrs[NDIRECT+1] = 0</code> (no master map), secondary map nonexistent, data block nonexistent.</p>

<p><strong>Step-by-step trace:</strong></p>
<pre><code class="language-c">// Incoming: bn = 267
// Subtract direct range:
bn -= NDIRECT;       // 267 - 11 = 256
// 256 &lt; NINDIRECT(256)? No → not singly-indirect.

bn -= NINDIRECT;     // 256 - 256 = 0
// 0 &lt; NDINDIRECT(65536)? Yes → doubly-indirect path.

// ── LEVEL 1: Master map ──
// ip->addrs[NDIRECT+1] = ip->addrs[12] = 0  → need to allocate
addr = balloc(ip->dev);         // (1) allocate master map block, e.g. block 500
ip->addrs[NDIRECT+1] = 500;     // store in inode (not yet on disk)

bp = bread(ip->dev, 500);       // (2) load the new (zeroed) master map block
a = (uint*)bp->data;            // a[] is all zeros

// idx1 = 0 / 256 = 0, idx2 = 0 % 256 = 0

// ── LEVEL 2: Secondary map ──
// a[0] = 0 → need to allocate secondary map
addr = balloc(ip->dev);         // (3) allocate secondary map block, e.g. block 501
a[0] = 501;                     // update master map buffer in RAM
log_write(bp);                  // (4) mark master map buffer dirty → survive crash

brelse(bp);                     // (5) release master map buffer BEFORE next bread()

// addr = 501, not 0, so we continue:
bp = bread(ip->dev, 501);       // (6) load the new (zeroed) secondary map block
a = (uint*)bp->data;            // a[] is all zeros

// ── LEVEL 3: Data block ──
// a[0] = 0 → need to allocate data block
addr = balloc(ip->dev);         // (7) allocate data block, e.g. block 502
a[0] = 502;                     // update secondary map buffer in RAM
log_write(bp);                  // (8) mark secondary map buffer dirty

brelse(bp);                     // (9) release secondary map buffer

return 502;                     // ← physical disk block for logical block 267</code></pre>

<p><strong>Summary — 9 operations for a fully cold doubly-indirect block:</strong></p>

<table>
  <tr><th>#</th><th>Call</th><th>Purpose</th></tr>
  <tr><td>1</td><td><code>balloc()</code></td><td>Allocate master map block</td></tr>
  <tr><td>2</td><td><code>bread()</code></td><td>Load master map into cache</td></tr>
  <tr><td>3</td><td><code>balloc()</code></td><td>Allocate secondary map block</td></tr>
  <tr><td>4</td><td><code>log_write(bp)</code></td><td>Mark master map dirty</td></tr>
  <tr><td>5</td><td><code>brelse(bp)</code></td><td>Release master map buffer</td></tr>
  <tr><td>6</td><td><code>bread()</code></td><td>Load secondary map into cache</td></tr>
  <tr><td>7</td><td><code>balloc()</code></td><td>Allocate data block</td></tr>
  <tr><td>8</td><td><code>log_write(bp)</code></td><td>Mark secondary map dirty</td></tr>
  <tr><td>9</td><td><code>brelse(bp)</code></td><td>Release secondary map buffer</td></tr>
</table>

<p>On subsequent writes to the same file, levels that already exist skip their <code>balloc()</code> calls. The warm path (all blocks already allocated, same secondary map) costs only: <code>bread(master)</code> → <code>brelse(master)</code> → <code>bread(secondary)</code> → <code>brelse(secondary)</code> → return. No <code>balloc()</code> or <code>log_write()</code> needed.</p>

</div>
</div>

<div class="var-question">
<div class="var-qnum">Q10 <span class="diff diff-hard">Hard</span></div>

<code>balloc()</code> returns 0 when the disk is full. The doubly-indirect path in <code>bmap()</code> calls <code>balloc()</code> up to three times. Write out every early-return path and explain what state is left on disk if allocation fails at each level.

<div class="answer-content">

<p><strong>The three failure points:</strong></p>
<pre><code class="language-c">// ── Failure at Level 1: master map allocation ──
if((addr = ip->addrs[NDIRECT+1]) == 0){
    addr = balloc(ip->dev);
    if(addr == 0)
        return 0;              // ← early return, disk full
    ip->addrs[NDIRECT+1] = addr;
}
// If we return here: ip->addrs[NDIRECT+1] is still 0.
// The inode is unchanged. No disk blocks were consumed.
// State: CLEAN — no leak, no corruption.

// ── Failure at Level 2: secondary map allocation ──
if((addr = a[idx1]) == 0){
    addr = balloc(ip->dev);
    if(addr){
        a[idx1] = addr;
        log_write(bp);
    }
}
brelse(bp);
if(addr == 0)
    return 0;                  // ← early return
// If we return here:
//   - Master map block IS allocated and stored in ip->addrs[NDIRECT+1].
//   - Master map entry a[idx1] = 0 (secondary map was never allocated).
//   - ip->addrs[NDIRECT+1] points to a real block with mostly-zero entries.
// State: SEMI-ALLOCATED — master map block is in use but points to nothing yet.
// The master map block is NOT leaked — it will be freed when itrunc() runs.

// ── Failure at Level 3: data block allocation ──
if((addr = a[idx2]) == 0){
    addr = balloc(ip->dev);
    if(addr){
        a[idx2] = addr;
        log_write(bp);
    }
}
brelse(bp);
return addr;                   // returns 0 if balloc failed
// If we return here:
//   - Master map IS allocated.
//   - Secondary map IS allocated, stored in master map.
//   - Data block was NOT allocated; secondary map entry is still 0.
// State: SEMI-ALLOCATED — both map blocks in use, pointing to no data block.
// writei() receives 0 from bmap() and handles the error upstream.</code></pre>

<p><strong>Are the semi-allocated states safe?</strong></p>
<p>Yes — because <code>itrunc()</code> walks the entire allocated tree and frees every block it finds, including partially built trees. When the file is deleted, <code>itrunc()</code> finds the master map block (non-zero <code>addrs[NDIRECT+1]</code>), iterates over its entries, frees any non-zero secondary maps and their data blocks, then frees the master map itself. No blocks are leaked regardless of which level the allocation failed at.</p>

</div>
</div>

<div class="var-question">
<div class="var-qnum">Q11 <span class="diff diff-hard">Hard</span></div>

The bigfile test writes <code>*(int*)buf = blocks</code> before each write. Why does it do this, and how does this allow the test to verify data integrity — not just that the file was large enough?

<div class="answer-content">

<p><strong>The write pattern:</strong></p>
<pre><code class="language-c">for (i = 0; i &lt; TEST_BLOCKS; i++){
    *(int*)buf = blocks;          // store current block index in first 4 bytes of buffer
    int cc = write(fd, buf, sizeof(buf));
    if(cc &lt;= 0)
        break;
    blocks++;
}</code></pre>

<p><strong>What this encodes:</strong></p>
<p>Each 1 KB buffer has its first 4 bytes set to the block's sequential index (0, 1, 2, ..., 6579). The rest of the buffer is uninitialized (contains whatever was in the variable on the stack before the loop), but the first <code>int</code> is deterministic and unique per block.</p>

<p><strong>How a read-back verification would work:</strong></p>
<pre><code class="language-c">// Hypothetical verification pass:
lseek(fd, 0, SEEK_SET);
for(i = 0; i &lt; TEST_BLOCKS; i++){
    read(fd, buf, sizeof(buf));
    int stored_index = *(int*)buf;
    if(stored_index != i){
        printf("data corruption at block %d: expected %d, got %d\n", i, i, stored_index);
        exit(-1);
    }
}</code></pre>

<p>A naive corruption would manifest as a block returning the wrong index — for example, block 300 returning index 44. This would indicate that <code>bmap()</code> returned the wrong physical block number for logical block 300, causing a read to return data that was actually written for logical block 44.</p>

<p><strong>What this detects beyond mere size:</strong></p>

<table>
  <tr><th>Bug type</th><th>How it would show up</th></tr>
  <tr><td>Wrong <code>idx1</code> or <code>idx2</code> calculation in bmap()</td><td>Logical blocks in the doubly-indirect range map to each other's physical blocks — reads return wrong indices</td></tr>
  <tr><td>Missing <code>brelse(bp1)</code> before <code>bread()</code> for bp2</td><td>Kernel panic (buffer exhaustion) — no output at all</td></tr>
  <tr><td>Missing <code>log_write()</code></td><td>Works during the run (buffer cache keeps the map in RAM), but a crash would corrupt the map — not detectable in a clean test run</td></tr>
  <tr><td>Off-by-one in range check (<code>NDIRECT</code> vs <code>NDIRECT+1</code>)</td><td>Block at the boundary (bn=11 or bn=267) maps to the wrong level — its data would return a wrong index</td></tr>
</table>

<p>The current <code>bigfile.c</code> only checks that <code>blocks == TEST_BLOCKS</code> after writing — it does not read back. But the write pattern is designed so that a read-back pass (not included in the grader) could detect mapping errors precisely.</p>

</div>
</div>
